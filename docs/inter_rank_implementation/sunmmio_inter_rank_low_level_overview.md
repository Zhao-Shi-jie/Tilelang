# sunmmio\_inter\_rank\_low\_level\_overview

# SunMMIO Rank 间通信底层机制概览

## 1\. 文档范围

本文简要整理 SunMMIO 当前 Rank 间通信的底层实现，重点说明接收完成 signal 的类型、
更新方式和顺序保证。结论来自 `compiler-samples` 当前 `main` 分支的 compute exchange、
ring all\-gather 以及公共 DMA/flagreg helper。

当前 TileLang 设计只考虑 compute core 发起的通信。Comm core 路径在本文中仅作为理解
DRAM memory signal 的参考，不属于当前最小闭环。

## 2\. 整体架构

一个 A4E Rank 包含 4 x 4 compute mesh，并按 mesh row 分成 4 个 sub\-device：

```Plaintext
sub-device 0: compute core 0..3
sub-device 1: compute core 4..7
sub-device 2: compute core 8..11
sub-device 3: compute core 12..15
```

对于 compute core：

```Plaintext
subdevice = core_id // 4
slot      = core_id % 4
```

RSRAM 和 DRAM 都属于 compute core 本地资源。TileLang TIR 使用 `global` scope 表示 DRAM
地址空间，但这里的 `global` 不是整个 Rank 共享内存：不同 compute core 具有各自独立的
DRAM buffer 实例。跨 row 访问另一个 core 的 DRAM 仍然需要明确的核间数据路径，这也是
kernel 可以按 core 使用 SPMD 方式执行的基础。

每个 sub\-device 有 north/south PCIe engine。当前 helper 为每个 engine 定义 4 个 write
channel 和 1 个 read channel；一个 channel 同时只能执行一条 descriptor chain。

Rank 间传输采用单边写。Sender 构造 PCIe descriptor，把本地数据主动写入 peer 的
buffer，receiver 不需要执行对称的 receive DMA。

## 3\. 对端地址

对端完整地址由三部分组成：

```Plaintext
peer address
  = peer sub-device aperture
  + compute/comm class 与 slot offset
  + peer local buffer address
```

其中 aperture 由 host/runtime 枚举并传给 kernel，peer local buffer address 由目标 Rank
实际分配后提供。不同 Rank 的 local buffer address 不能假设相同。

Compute core 地址由 helper 组合：

```C++
su_pcie_compute_addr(peer_aperture, peer_slot, peer_local_addr)
```

Descriptor 的 payload 地址属于 PCIe aperture 地址域；descriptor chain 自身的地址属于
chip\-scoped physical address 域，两者不能混用。

## 4\. Doorbell 与 Signal

源码中有几个容易混淆的概念：

|名称|作用|
|---|---|
|PCIe channel doorbell|Sender 本地启动 DMA engine|
|Channel sync flag/ISF|结束 descriptor chain，形成 sender 本地 `STOPPED`|
|Receiver signal|通知接收方 payload 已经可见|

前两项属于 DMA backend 内部状态。TileLang `T.dist` 需要表达的是第三项。

### 4\.1 Receiver Signal 的六种形式

|Signal 类型|每个 compute core 数量|更新语义|多 sender|
|---|---|---|---|
|SRAM increment flagreg|8|远端写自动 `+1`|允许聚合|
|DRAM increment flagreg|8|远端写自动 `+1`|允许聚合|
|SRAM value flagreg|32|覆盖写入明确 generation|每 sender 独立 signal|
|DRAM value flagreg|32|覆盖写入明确 generation|每 sender 独立 signal|
|SRAM memory signal|基本不限|覆盖写入明确 generation|每 sender 独立 signal|
|DRAM memory signal|基本不限|覆盖写入明确 generation|每 sender 独立 signal|

六类 signal 的物理地址、value 位宽和 wait helper 需要随底层接口继续补齐。当前文档中已验证
的固定 offset 和 `uint8` 回绕规则主要对应 increment flagreg；不能直接外推到 value
flagreg 和 memory signal。

### 4\.2 Increment Flagreg

Flagreg 的固定 local offset 为：

```C++
SU_FLAG_SRAM(n) = 0x0D000000 + n * 0x800
SU_FLAG_DRAM(n) = 0x3C0000000 + n * 0x800
```

`n` 的合法范围为 `0..7`。Sender 通过目标 Rank 的 aperture、目标 slot 和这个 offset，
可以指定更新对端 core 上的某一个 flagreg：

```C++
peer_flag = su_pcie_compute_addr(peer_aperture, peer_slot, SU_FLAG_SRAM(n));
```

向该地址执行一次普通 PCIe/ODMA write，会让对应的 8\-bit flagreg 自动 `+1`。样例使用
一次 4 字节写；写入值本身会被丢弃，因此 source 可以是任意可读的 4 字节数据。

写普通 RSRAM 或 DRAM payload 不会自动更新 flagreg。只有写入 flagreg 的固定地址才会
触发 `+1`。连续两次写同一个 flagreg 地址才会使它增加 2。

Receiver 通过本地 CSR 等待 flagreg 到达 expected generation。比较支持 8\-bit 回绕，
但 waiter 不能落后超过 127 次更新。等待成功后执行 acquire fence，再读取 payload。

### 4\.3 Value Flagreg 与 Memory Signal

Value flagreg 和 SRAM/DRAM memory signal 都不具备自动 `+1` 语义。Sender 必须写入明确
generation，例如 `1、2、3...`。由于多个 sender 覆盖写同一位置会产生竞争，每个
receiver endpoint 上的每个 sender 必须使用独立 signal。

DRAM memory signal 是普通 DRAM buffer，不具备硬件自动 `+1` 语义。Host/runtime 负责：

```Plaintext
1. 分配 signal buffer
2. 初始化初始值，当前样例为 0
3. 把本地和 peer signal 地址传给 kernel
```

Sender 需要明确写入 generation，例如 `1、2、3...`。Receiver 从 memory 中读取该值并
判断 `observed >= expected`。写入普通 memory signal 不会更新任何 SRAM/DRAM flagreg。

当前 ring all\-gather 样例为每个 compute core 分配 256 B signal slot，但逻辑上只使用
第一个 32\-bit word。256 B 来自 comm\-core vector width 和 ODMA 最小传输粒度，不是
signal 语义要求。

## 5\. Put 与 Descriptor Chain

底层没有单独的 `put` 指令封装。一次 put 由以下步骤组成：

```Plaintext
1. 组合本地和对端 PCIe 地址
2. 为 payload 构造一个或多个 descriptor
3. 在 chain 末尾追加 receiver signal descriptor
4. 选择 PCIe write channel 并提交 chain
```

Descriptor 通过 `next` 指针组成有序 chain，DMA engine 按顺序执行：

```Plaintext
descriptor 0..N-1: 写入 payload
descriptor N:      写入 receiver signal
```

因此 receiver signal 只会在 payload 完成后更新。TileLang 的 `put` 应当把 payload 和
signal 保持为一个完整操作，不向用户暴露独立的 `send_payload` 和 `send_signal`。

对于 increment flagreg，最后一个 descriptor 向固定 flagreg 地址执行写入，使其自动
`+1`。对于 value flagreg 和 memory signal，最后一个 descriptor 向对应 signal 地址写入
明确 generation。

最后一个 descriptor 还可以设置 `ISF`，但 `ISF` 更新的是 sender DMA channel 的内部
sync flag，不会让 receiver flagreg 再增加一次。

## 6\. Wait 的实现

底层 wait 都是轮询，但等待对象不同：

|Wait 类型|轮询对象|完成含义|
|---|---|---|
|Sender wait|本地 DMA channel status|Source 和 descriptor 可以复用|
|Increment flagreg wait|本 core 指定 flagreg CSR|对应数量的 payload 已可见|
|Value flagreg wait|本 core 指定 value flagreg|对应 generation 的 payload 已可见|
|Memory signal wait|本地 signal memory 中的 generation|对应 payload 已可见|

Sender wait 已封装为 `su_pcie_dma_wait(channel)`。它反复读取 channel status，直到状态
不再是 `RESERVED/RUNNING`，随后执行 fence；调用者还需要检查最终状态为 `STOPPED`。

Flagreg wait 已封装为 `su_flag_wait_local(csr, expected)`。它只轮询指定的一个 CSR，
不是遍历 8 个 flagreg；观察到 generation 后执行 acquire fence。

普通 DRAM memory signal 没有公共 wait helper。Ring all\-gather 样例由 comm core 循环
使用 ODMA 把多个 signal 搬到 RSRAM，遍历检查 `observed >= expected`，未全部完成则
重新搬运和检查。Compute core 如何等待普通 DRAM signal 尚无验证样例。

当前底层也没有通用的 `wait_signal_list` 或 `wait_any_signal`，后续需要由 SUVM/backend
提供或展开为轮询逻辑。

## 7\. 对 TileLang 设计的直接含义

- `put` 必须把 payload 和 receiver completion 绑定为一个有序通信语义。

- Signal TIR 需要区分 SRAM/DRAM scope、increment/value flagreg 和 memory signal。

- Increment flagreg 允许多个 sender 聚合；value flagreg 和 memory signal 必须绑定唯一 sender。

- Flagreg signal 需要记录 bank、更新模式、index、owner endpoint 和 generation。

- Memory signal 需要记录 buffer region、owner endpoint 和写入的 generation value。

- 本地 send completion 与远端 receiver signal 必须分开建模。

- 自动 kind 的 INC/VALUE/MEMORY 选择优先级需要结合 sender 拓扑、资源容量和底层 wait
能力进一步确认。

- Value flagreg、SRAM memory 和 compute\-core DRAM memory 的写入位宽及 wait helper 仍需
额外验证。

## 8\. 参考源码

- `samples/31_compute_exchange/kernel.cpp`

- `samples/33_ring_allgather_dram/kernel.cpp`

- `common/su_pcie_dma.h`

- `common/su_flag_reg.h`

- `common/su_core_placement.h`

- `sunsim/src/sunsim/commgroup.py`

- `sunsim/src/sunsim/topology.py`

