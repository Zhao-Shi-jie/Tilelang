# sunmmio\_inter\_rank\_communication\_design

# SunMMIO Rank 间通信设计大纲

## 文档状态

- 状态：阶段三静态 P2P 已收口，Collective lowering 架构已确认、具体 op 待逐项设计

- 版本：0\.19

- 最后更新：2026\-09\-01

- 当前开发边界：前端语法与后端 TIR 处理

本文档用于约束本轮 Rank 间通信开发的范围和推进顺序。`testing/my_test` 下的草稿仅作
参考；经过讨论确认的设计会逐步补充到本文档中。

## 1\. 本轮目标与边界

### 1\.1 本轮包含

本轮完成从 TileLang DSL 到最终 device kernel TIR 的完整处理：

```Plaintext
T.dist 前端语法
  -> 高层分布式 TIR
  -> 语义检查与参数归一化
  -> 分布式通信 lowering
  -> host/device split 及后续 TIR pass
  -> 最终 device kernel TIR
```

具体包含：

- 搭建 `T.dist` 基础语法和公共对象。

- 定义 Rank、world、endpoint、signal 和通信操作的 TIR 表示。

- 在 `LowerAndLegalize` 与 `OptimizeForTarget` 中加入所需的检查和 lowering。

- 完成 `put + signal + wait_signal` 最小闭环。

- 扩展不同 endpoint、buffer region、signal 和通信模式。

- 支持多种 collective op 的前端语义和 TIR 处理。

- 建立 DSL、逐 pass TIR、最终 device TIR 和错误场景测试。

### 1\.2 本轮不包含

- SUVM codegen 对接。

- 从 TIR 生成 SUVM MLIR 或硬件指令。

- Runtime launch、远端地址注册和真实设备 ABI 对接。

- 模拟器或真实硬件执行验证。

- 依赖 SUVM 实现细节的性能调度和资源分配。

本轮最终产物是稳定、可检查的 device kernel TIR。后续 SUVM 支持就绪后，再基于这层
TIR 契约实现 codegen 及其后的流程。

## 2\. 已确认的设计方向

1. Rank 间通信使用新的 `T.dist` 命名空间，现有 `T.comm` 保持 Rank 内语义。

2. 用户使用逻辑 `(rank, core)` endpoint，具体物理路径不进入前端语义。

3. 复用现有 Buffer 和 MeshTensor，不新增独立的分布式 Tensor 类型。

4. 第一版使用名为 `world_size` 的普通 Python `int` 外层编译参数；默认值为 1，表示
不启用 Rank 间通信。显式传入时它是编译缓存 key 的一部分，并保存为 PrimFunc 属性。

5. `rank_id` 是显式的运行时 `int32` PrimFunc 参数，并通过语义标记识别，不能只依赖参数名。

6. `T.Kernel()` 仍然只表示当前 Rank 内的 core mesh，不承载 Rank 维度。

7. MeshTensor 使用独立的 `rank_placement` 和现有 core `placement`，按 Rank、core 两层依次切分。

8. MeshTensor shape 使用 `global_shape -> rank_shape -> local_shape` 三级模型。

9. 第一版使用显式 signal、receiver wait；sender wait 由编译器
根据 source 生命周期自动插入，并从公共写法中隐藏。

10. 远端 signal 与现有 Rank 内 `tl.sync_token_id` 采用不同的语义和 TIR。

11. Collective 建立在统一的 endpoint、buffer region 和完成事件模型之上。

## 3\. 核心设计模块

### 3\.1 前端语法

Rank/world 基础语法采用以下约束：

- `world_size` 是外层 kernel factory 中名为 `world_size` 的普通 Python `int` 参数，
不增加 `T.dist.WorldSize` 标记类型；省略时按 1 处理。

- 显式传入的 `world_size` 必须是正整数，`bool` 不视为合法整数。

- 现有 JIT 参数机制自动将 `world_size` 纳入 phase\-1 编译缓存 key。

- JIT 在构造 PrimFunc 期间，根据外层 `world_size` 参数建立内部 Rank 编译上下文。
MeshTensor 和 `T.dist.world_size()` 从该上下文读取同一个值，用户不需要在每个
MeshTensor 上重复传入 `world_size`。

- `world_size` 保存为 PrimFunc 属性 `tl.dist.world_size`，类型为 `int32 IntImm`。

- `T.dist.world_size()` 直接返回该编译期 `IntImm`，第一版不生成待后续 pass 解析的
world size symbol。

- 当前不存在 Rank 编译上下文时，`rank_placement` 和 `T.dist.world_size()` 使用默认
`world_size=1`。

- 一个 PrimFunc 只允许一个 `world_size`，所有 MeshTensor 必须使用相同的编译上下文。

- `rank_id` 由 kernel 启动时传入，是显式的运行时 `int32` PrimFunc 参数。

- 第一版使用 `T.dist.RankId` 专用参数注解；用户参数名可以不是 `rank_id`。

- Frontend 将该注解降为普通 `int32` PrimFunc 参数，并通过 PrimFunc 属性记录唯一的
参数索引，例如 `tl.dist.rank_id_param_index`，后续 pass 不依赖参数名称匹配。

- 一个 PrimFunc 最多只能有一个 `T.dist.RankId` 参数。

- 本轮只保证 `rank_id` 保留到最终 device TIR，不实现 launcher 传参。

- `with T.Kernel() as core_id` 继续只返回当前 Rank 内的 core ID。

第一版推荐写法为：

```Python
@tilelang.jit(target="sunmmio")
def build_kernel(M, N, world_size: int = 1):
    @T.prim_func
    def main(
        A: T.MeshTensor(
            (M, N),
            placement=T.placement.row_shard(0),
            rank_placement=T.dist.placement.shard(0),
            dtype=T.bfloat16,
        ),
        rank_id: T.dist.RankId,
    ):
        with T.Kernel() as core_id:
            ...

    return main

kernel = build_kernel(M, N, world_size=4)
```

后续可以增加 `T.dist.WorldSize` 标记以摆脱参数名约定，但不属于第一版。

其他需要逐步确定的语法包括：

- Endpoint 的表达方式及默认 core 规则。

- Signal 和 signal list 的创建方式。

- P2P op 的函数签名和参数规则。

- Wait op 的函数签名和返回值。

- Collective op 的函数签名、shape 规则和参与范围。

- Buffer、BufferRegion、动态表达式和错误参数的归一化行为。

本模块产出 Python DSL、TIR script printer 表示以及前端单元测试。

### 3\.2 MeshTensor Rank Placement

MeshTensor 增加独立的 `rank_placement` 参数，现有 `placement` 继续表示 Rank 内的
二维 core mesh placement。两个参数不复用同一个类型，但保持相似的使用风格：

```Python
A: T.MeshTensor(
    (M, N),
    placement=T.placement.row_shard(0),
    rank_placement=T.dist.placement.shard(0),
    dtype=T.bfloat16,
)
```

Rank placement 第一阶段只需要两种形式：

- `replicated()`：每个 Rank 都具有完整的声明 shape；

- `shard(dim)`：沿指定 Tensor 维度在 `world_size` 个 Rank 间切分。

省略 `rank_placement` 等价于 `replicated()`。这里的 replicated 只描述物理 shape，
不保证不同 Rank 上的数据内容相同。

Shape 按以下顺序计算：

```Plaintext
global_shape
  -> rank_shape，由 rank_placement 和 world_size 计算
  -> local_shape，由现有 core placement 和 mesh shape 计算
```

`shard(dim)` 使用统一物理容量：

```Plaintext
rank_shape[dim] = ceildiv(global_shape[dim], world_size)
```

当 shape 不能整除时，`rank_shape` 和 `local_shape` 表示统一物理容量，实际有效范围依赖
`rank_id` 和 `core_id`。MeshTensor 至少需要提供：

```Python
A.global_shape
A.rank_shape
A.local_shape
A.get_rank_extent(rank_id)
A.get_local_extent(core_id, rank_id=rank_id)
```

现有 `get_local_extent(core_id)` 保持兼容；仅当 Tensor 使用 Rank shard 时，才要求传入
`rank_id`。

`tensor_meta` 至少增加以下信息：

```Plaintext
global_shape
rank_shape
local_shape
world_size
rank_placement
placement
mesh_shape
```

现有 `tensor_meta.global_shape` 还参与 public buffer 校验，而 Rank shard 后每个 Rank
实际持有的物理容量通常是 `rank_shape`。实现前必须明确相关 pass 使用 global shape
还是 rank shape，不能通过重载 `global_shape` 的含义来兼容。

第一版确定如下：

- `global_shape` 始终表示跨 Rank 的逻辑 shape，不改变现有字段名称和含义。

- `rank_shape` 表示单个 Rank 启动时持有的统一物理 shape。

- TIR 中实际参数 Buffer 仍使用经过 core placement 后的 `local_shape`。

- 需要重建或校验当前 Rank 公共参数物理 shape 的 pass 使用 `rank_shape`；未设置
`rank_placement` 时 `rank_shape == global_shape`，现有行为不变。

- Layout 推导必须分别保留 global、rank 和 local 层次，不能把 `rank_shape` 写回
`global_shape`。

### 3\.3 Signal 与完成语义

所有 Rank 间 signal API 保留在 `T.dist` 命名空间。单个和多个 signal 的公共签名为：

```Python
signal = T.dist.signal(kind=None)
signals = T.dist.signals(count, kind=None)
```

用户省略 kind 时，frontend 使用 `None` 表示自动规划，因此常规代码直接使用：

```Python
signal = T.dist.signal()
signals = T.dist.signals(4)
```

用户显式选择时必须能够选择完整物理类型，因此 frontend 保留完整枚举：

```Python
T.dist.SignalKind.SRAM_FLAGREG_INC
T.dist.SignalKind.DRAM_FLAGREG_INC
T.dist.SignalKind.SRAM_FLAGREG_VALUE
T.dist.SignalKind.DRAM_FLAGREG_VALUE
T.dist.SignalKind.SRAM_MEMORY
T.dist.SignalKind.DRAM_MEMORY
```

`kind=None` 表示由编译器推断完整六选一 kind；用户显式指定枚举时，编译器不得改变
kind，只进行 scope、容量、sender 和使用合同检查。Frontend 继续使用枚举以避免字符串
拼写错误；TIR 不使用数字 kind，而使用稳定、可读的 `StringImm` 名称：

```Plaintext
sram_flagreg_inc
dram_flagreg_inc
sram_flagreg_value
dram_flagreg_value
sram_memory
dram_memory
```

初始 TIR 中，省略 kind 表示为 `dist_signal_decl("auto", logical_id)`；显式 kind 使用对应
机制全名。`PlanDistSignals` 处理后统一得到
`dist_signal("sram_flagreg_inc", index)` 这类 resolved signal。

Signal index 不暴露给用户。Frontend 只分配稳定的 logical declaration id，
`PlanDistSignals` 在 scope 确定后按 kind 分配最终 index。
`T.dist.signals` 第一版接受正的编译期整数 count，返回 `SignalList`。每个元素仍是一个
独立 signal declaration，仅支持 `signals[0]` 这类编译期 Python 整数索引；TIR expression
动态索引留到需要动态 signal 选择的协议中再设计。

六种 signal 的资源和更新规则为：

|Resolved kind|更新方式|每个 endpoint 容量|Sender 规则|
|---|---|---|---|
|`sram_flagreg_inc`|硬件自动 `+1`|8|允许多个 sender 共享|
|`dram_flagreg_inc`|硬件自动 `+1`|8|允许多个 sender 共享|
|`sram_flagreg_value`|Sender 直接写 generation|32|每个 sender 使用独立 signal|
|`dram_flagreg_value`|Sender 直接写 generation|32|每个 sender 使用独立 signal|
|`sram_memory`|Sender 直接写 generation|基本不限|每个 sender 使用独立 signal|
|`dram_memory`|Sender 直接写 generation|基本不限|每个 sender 使用独立 signal|

- 每个 endpoint 都具有相同 signal declaration 对应的本地实例。

- `flagreg_inc` 每完成一次物理 put 由硬件原子自增一次；receiver expected 等于所有
sender 物理 put 次数之和，因此多个 sender 可以聚合到同一 signal。

- `flagreg_value` 和 `memory` 都由 sender 写入显式 generation。对同一个 receiver
endpoint，一个 signal index 只能绑定一个 physical sender；同一个 sender 可以使用该
signal 连续执行多次 put，不同 sender 必须使用不同 signal。

- Physical sender 定义为 `(src_rank, src_row, current_col)`；唯一 sender 约束按
`(receiver endpoint, signal index)` 检查。

- `wait_signal(s, ...)` 等待当前 expected generation，本身不修改 generation；重复 wait
会等待同一个值。

- `wait_all(signals, dst=...)` 等待列表中每个 signal 的当前 expected，第一版在
`LowerDistCommunication` 中按声明顺序展开为多个 `wait_signal` leaf，不增加 backend leaf。

- Function 属性 `tl.dist.signal_counts` 调整为以上六个稳定字符串到数量的具名 Map，并
保留到最终 device kernel TIR；旧三元素数组和 `tl.dist.sram_signal_count` 兼容属性在
实现调整时统一移除。

Signal scope 严格绑定 receiver destination scope，不由 source scope 决定：

- `shared.rsram` destination 只能使用三种 `SRAM_*` kind。

- `global`/DRAM destination 只能使用三种 `DRAM_*` kind。

- 同一个 signal 的所有 put/wait destination 必须属于同一 scope；混用直接报错。

- 显式 kind 与 destination scope 不一致时直接报错，不进行隐式替换。

- `kind=None` 时，编译器先从 destination 确定 signal scope，再结合 sender 拓扑和资源
容量推断完整 kind。具体推断优先级必须在实现前单独确认，不能用会改变多 sender
合法性的静默降级策略。

Generation 在最终 TIR 中显式表示，但不暴露在 Python API 和初始高层 TIR 中。
`LowerDistCommunication` 为每个被使用的 signal 创建 sender generation 和 receiver
expected state。稳定 put leaf 继续统一携带 `send_generation`：`flagreg_inc` 的 backend
忽略该值，`flagreg_value/memory` 使用该值。当前统一 `uint8` state 必须改为由 resolved
kind 决定；自增 flagreg 的回绕规则、value flagreg 和 memory 的写入位宽及 wait 比较规则
需要在实现前依据底层接口确认。
`LowerDistCommunication` 的内部 expectation planning 阶段根据实际 peer route 为
destination 生成紧凑的 `dist_expect` marker。它可在编译期枚举 Rank 计算入边，但不在
device TIR 中复制 Rank 路由。
`InjectDistSync` 在 pipeline 规划和 host/device split 之后，在每个物理 peer put 前推进
对应 sender generation，并将 `dist_expect` 改写为 expected 更新。`flagreg_inc` 的
expected 汇总所有 sender contribution；`flagreg_value/memory` 在唯一 sender 前提下维护
显式 generation。wait 只读取 expected，重复 wait 不修改它。

完成事件分为两类：

- `T.dist.wait_signal(signal, dst=...)`：receiver completion；返回后 `dst` payload 已可见。

- `T.dist.wait_all(signals, dst=...)`：依次满足列表中全部 receiver completion；所有 signal
共用一个覆盖完整 payload 的 destination region 作为数据依赖。

- `T.dist.wait()`：sender local completion；返回后此前 put 的 source 可以复用。

第一版两者都显式书写。后续允许编译器根据 source buffer 生命周期自动插入
`T.dist.wait()`，但不能用 receiver signal wait 替代 sender completion。

### 3\.4 P2P 最小操作

第一版公共语法为：

```Python
T.dist.put(src, dst, dst_rank=peer_rank, dst_row=None, signal=signal)
T.dist.routed_put(
    src,
    dst,
    routes=[[src_row, dst_rank, dst_row], ...],
    signal=signal,
    src_rank=None,
)
T.dist.wait_signal(signal, dst=dst)
T.dist.wait_all(signals, dst=dst)
T.dist.wait()
```

约束如下：

- 仅支持 compute core 发起通信。

- Rank 间 kernel 习惯上使用 `T.MeshTensor` tensor 参数；第一版不增加专门的强制校验。

- 数据 scope 支持四种组合：`RSRAM -> RSRAM`、`RSRAM -> DRAM`、
`DRAM -> RSRAM` 和 `DRAM -> DRAM`。TIR 中 DRAM 使用 `global` scope，但它是每个
compute core 独有的 core\-local 地址空间，不是整个 Rank 共享的内存。Signal kind
仍只由 receiver destination scope 决定，因此 `DRAM -> RSRAM` 使用 SRAM signal。

- `dst_row=None` 表示远端对等 row；显式 `dst_row` 只改变 row，column 保持不变。

- 本轮 frontend endpoint 只描述 Rank 和 row，不接收 `src_column`、`dst_column`、
`src_core` 或 `dst_core`。Source/destination column 始终隐式等于当前执行 column，
routing 只允许在同一 column 内选择或转发 row。外层 `current_col` 条件仍可作为普通
SPMD 控制流存在，但不改变 route endpoint 的 column。

- 对等 route 不展开 payload；存在 cross\-row route 时，compiler 枚举所有 source row，
先用 `T.comm.put` 把 payload 转发到本 Rank egress row，再由 egress 执行对等 Rank put。

- 需要显式选择 source row 时推荐使用 `T.dist.routed_put`。Route table 始终保持三字段
`[src_row, dst_rank, dst_row]`，一次调用中的所有 route 共用调用级 `src_rank`。

- `src_rank=None` 表示当前执行 `rank_id`，保持现有 SPMD 语义，不生成额外 Rank 条件。
显式 `src_rank=n` 第一版要求 `n` 是范围 `[0, world_size)` 内的编译期静态整数，
`LowerDistRouting` 将其规范化为 `if rank_id == n` 包围的内部三字段 routed put。

- 不同 source Rank 使用多个 `routed_put` 调用；每个调用仍可包含多条 row route，不要求
将 route table 展开为 `world_size` 份。不同 source Rank 写入同一 receiver endpoint 时，
仍需使用可区分的 signal。

- `routed_put` 的外层条件必须对同一 Rank/current column 的所有 row 一致。
`rank_id` 和 `current_col` 条件允许；`current_row` 或完整 `cid` 条件不允许。

- 指定单个 cid 时，将其拆为外层 `current_col` 条件和 route table 中的 `src_row`。

- `put` 是异步单边写，必须携带 receiver signal。

- `wait_signal` 的 `dst` 第一版为必填参数，用于建立明确的本地数据依赖。

- `wait_all` 第一版要求一个覆盖全部相关 payload 的 `dst`，并展开为多个单 signal wait。

- `wait` 等待当前 endpoint 之前提交的全部本地发送完成。

- BufferRegion 支持固定长度切片和非零/表达式 offset；extent 第一版必须是编译期常量。

- `dst_rank == rank_id` 的 route 不进入 Rank 间发送：同 row route 改写为 `T.copy`；
cross\-row RSRAM destination 直接使用 `T.comm.put`；cross\-row DRAM destination 先
`T.comm.put` 到目标 core 的 RSRAM staging，再由目标 core copy 到自己的 DRAM。

- 每个 cross\-row routed put 使用独立 compiler staging，不跨 routed put 复用。用户负责显式
source/staging 的生命周期，并在复用前显式调用 `T.dist.wait()`。

- 显式 route 检查 source/destination endpoint 范围、完全重复表项，以及同一 receiver
signal 是否存在多个物理 sender。不同 payload region 的重叠由用户负责。

- TIR 已支持自动 SRAM/DRAM flagreg 和显式 DRAM memory signal 规划；暂不实现
这些 kind 到 SUVM 的 codegen、跨 column、接收侧转发和 proxy 路由。后续即使扩展
column/core endpoint，也不改变本轮 row\-only API 的语义。

- Cross\-row 自动路由当前只支持 flagreg signal；SRAM/DRAM memory signal 的地址和 wait
路径留待底层接口确认。

- Cross\-row put 不允许出现在用户循环中，直到 staging 跨迭代生命周期得到完整设计。

- BufferLoad、source\-local count/valid 或数据驱动目标不属于静态 routed put，后续使用动态
route metadata 或 all\-to\-allv 协议。

一次 `put` 在 TIR 中是不可拆分的有序语义：

```Plaintext
写入远端 payload
  -> payload 对 receiver 可见
  -> 更新 receiver signal
```

PCIe channel doorbell、ISF 和 descriptor chain 属于未来 SUVM codegen 内部实现，不进入
公共 DSL 或高层 TIR。

### 3\.5 分布式 TIR

需要定义两层 TIR：

- 高层分布式 TIR：保留逻辑 endpoint、BufferRegion、signal 和 collective 语义。

- 最终 device TIR：完成必要的标准化和展开，可供未来 SUVM codegen 消费。

需要逐步确定：

- 每项操作的 op 名称、参数顺序、类型和副作用。

- Rank/world symbol 在 PrimFunc 中的表示。

- Signal 对象在 TIR 中的表示和生命周期。

- 远端 buffer identity 需要保留的最小元数据。

- 哪些高层操作在 TIR 阶段展开，哪些保留为底层 leaf op。

- 哪些函数或 buffer 属性必须跨越 host/device split。

第一版高层 op：

```Plaintext
tl.dist_signal_decl
tl.tileop.dist_put
tl.tileop.dist_peer_put
tl.tileop.dist_wait_signal
tl.dist_wait_all
tl.dist_expect
tl.dist_wait_send
tl.dist_rank_routed_put
```

Cross\-row 路由使用两级显式中间 op。第一级为：

```Plaintext
dist_rank_routed_put(src, dst, routes, src_rank, signal, current_core)

dist_route_table([
    [origin_src_row, dst_rank, dst_row],
    ...
])

dist_routed_put(src, dst, routes, signal, current_core)
```

公共 `T.dist.routed_put` 生成 `dist_rank_routed_put`。其中 `src_rank` 始终具有同一语义：
省略时 frontend 写入当前 `rank_id`，显式指定时写入静态 Rank。`LowerDistRouting` 将当前
Rank 情况直接改为内部 `dist_routed_put`；将静态 Rank 情况改为
`if rank_id == src_rank` 包围的内部 `dist_routed_put`。因此内部 route table 继续只描述
row 和 destination，不携带 source Rank。

`dist_routed_put` 只表示本 Rank 内尚待执行的 row\-routing，不发送 Rank 间数据，
也不推进 signal。Signal 只作为 completion 绑定传递到第二级。本 Rank 转发后为：

```Plaintext
dist_peer_route_table([
    [peer_row, dst_rank],
    ...
])

dist_routed_peer_put(peer_sources, remote_destinations, routes, signal,
                     current_core)
```

`peer_row` 同时是本地实际核外发送 row 和远端 destination row，因此每条物理 route
满足 `physical src_row == remote dst_row`。`dist_routed_peer_put` 才是实际 Rank 发送操作。

第一版最终 device leaf op：

```Plaintext
tl.dist_put_(
    src_region,
    remote_dst_region,
    dst_rank,
    signal_kind,
    signal_index,
    send_generation,
)

tl.dist_wait_signal_(
    signal_kind,
    signal_index,
    expected_generation,
    local_dst_region,
)

tl.dist_wait_send()
```

第一版 public API 只接收 row，不接收 column 或完整 core endpoint。Frontend 把当前
`core_id` 作为高层 op 的内部参数，只用于获得 `current_row/current_col`；路由改变 row 时
始终保留 `current_col`。`LowerDistRouting` 内部依次
分离本 Rank route、将远端 cross\-row put 规划为三字段 route table 和
`dist_routed_put`，再生成发送 Rank 内 `T.comm.put` 和两字段 `dist_routed_peer_put`。
高层 `dist_signal_decl(requested_kind, logical_id)` 返回的 handle 只用于建立 frontend
signal declaration 与 put/wait 的引用关系。`PlanDistSignals` 将 declaration 改写为
`dist_signal(kind, index)`，明确表示已完成资源规划的 signal。`LowerDistRouting` 和
`LowerDistCommunication` 只接受规划后的 `dist_signal`；后者再将
`kind/index/send_generation/expected_generation` 写入对应 leaf，并消除 signal handle。
最终 device TIR 不保留 `dist_signal_decl` 或 `dist_signal`。

`dist_wait_all(dst, signals...)` 只存在于高层 TIR。`LowerDistCommunication` 将其展开为
多个稳定 `dist_wait_signal_` leaf，因此最终 device TIR 和未来 SUVM 接口不新增 wait\-all op。

`dist_put_` 保留 sender generation，`dist_wait_signal_` 保留 receiver expected。从
`LowerDistCommunication` 开始两个 leaf 的参数数量和顺序不再改变。后续 codegen 根据
`signal_kind` 决定是使用还是忽略 put 的 send generation：flagreg 由硬件自动递增，
DRAM memory signal 则需要显式写入该值。

### 3\.6 语义检查与归一化

需要覆盖：

- Rank/core 范围和 endpoint 表达式合法性。

- `world_size` 的有效值必须是非 `bool` 的正整数；缺省值为 1。

- `world_size=1` 且没有 dist op 时，各 dist pass 直接跳过；若出现 dist 通信 op，
`PlanDistSignals` 的内部 validation 阶段给出明确错误。

- `rank_id` 参数必须由 `T.dist.RankId` 声明，降级后为 `int32`，并具有唯一参数索引标记。

- Rank placement 维度、三级 shape 和非整除有效范围的合法性。

- `dst_row` 的默认值、静态可分析性和 endpoint 范围。

- Source/destination dtype、shape、scope 和 region 合法性。

- 固定 extent region 与非零/表达式 offset；静态有界动态 extent 留待动态通信协议。

- 显式 route endpoint 范围、完全重复表项，以及按 resolved kind 检查多 sender 合法性。

- 同一 destination region 的一般多写者/alias 冲突由用户负责，第一版不做完整 alias 分析。

- Signal scope 必须与 receiver destination scope 完全一致；同一 signal 的所有 put/wait
destination 必须属于同一 scope class。

- SRAM/DRAM 自增 flagreg 分别不能超过 8，value flagreg 分别不能超过 32；memory 使用
compiler logical slot，物理资源由后续 runtime 约束。

- 显式六选一 kind 必须与 destination scope 兼容；自动 kind 必须最终解析为完整字符串 kind。

- 第一版 `put` 必须携带 signal，`wait_signal` 必须携带 local destination region。

- `flagreg_inc` 允许多个物理 sender 聚合；`flagreg_value/memory` 要求每个 receiver
signal 具有唯一物理 sender。

- 显式 sender wait 与 source buffer 复用顺序由用户保证；自动插入留给后续独立 pass。

- Collective 的参与一致性、shape 和 signal 数量。

- 不支持能力必须在 codegen 前给出明确错误。

### 3\.7 TIR Lowering 与 Pass

正式生产 pipeline 只注册四个 dist pass。每个 pass 对应一个独立 `*dist*.cc` 文件，
并像现有 SunMMIO pass 一样在内部组织 analysis、validation、planning 和 rewrite 阶段：

1. `PlanDistSignals`

1. 文件为 `src/transform/plan_dist_signals.cc`，位于 `InferSramScope` 之后。内部先汇总每个
signal 的 put/wait destination scope，检查 scope 一致性和用户 kind 约束，按 kind
分配资源并改写 resolved `kind/index`；随后检查高层 dist op 的 signal、scope、dtype、
静态 region 和基础参数合同。没有 dist op 时直接返回原 IRModule；单 Rank kernel 出现
dist op 时在此报错。

2. `LowerDistRouting`

2. 文件为 `src/transform/lower_dist_routing.cc`，位于 `PlanDistSignals` 之后、
`LegalizeSunmmioDataPath` 之前。内部依次执行：

    - 规范化 `dist_rank_routed_put`：省略 source Rank 时保持当前 Rank SPMD 语义；显式
    source Rank 时生成 Rank guard，并转为内部三字段 `dist_routed_put`。

    - 分离本 Rank route：同 row 生成 copy，cross\-row RSRAM destination 生成
    `T.comm.put`，cross\-row DRAM destination 使用目标 RSRAM staging 和 writeback。

    - 检查 endpoint、静态 route、外层条件和重复表项；peer route 生成
    `dist_peer_put`，cross\-row route 生成三字段 `dist_routed_put`。

    - 消费 `dist_routed_put`，为 RSRAM/DRAM core\-local source 创建独立 RSRAM staging，
    生成 Rank 内 `T.comm.put` 和两字段 `dist_routed_peer_put`。

2. 该 pass 不修改 signal state。生成的本地 copy/comm 统一交给随后现有的
`LegalizeSunmmioDataPath` 处理。

3. `LowerDistCommunication`

3. 文件为 `src/transform/lower_dist_communication.cc`，位于 `SunmmioLayoutInference` 之后、
原有 `LowerTileOp` 之前。内部先根据 direct/routed peer route 按
`(dst_rank, peer_row, signal)` 汇总 receiver expectation；允许 `flagreg_inc` 聚合多个
sender，要求 `flagreg_value/memory` 具有唯一 sender，并生成内部 `dist_expect` marker；
随后为 signal 分配 sender
generation 和 receiver expected state，将 peer put、wait、wait\-all 和 expectation
一次性改写为稳定 leaf，并消除 signal handle。

3. 原有 `LowerTileOp` 不增加 dist 特判；运行到它时 dist TileOperator 已处理完成。

4. `InjectDistSync`

4. 文件为 `src/transform/inject_dist_sync.cc`，位于 `OptimizeForTarget` 中，在 pipeline 规划、
host/device split 和 `MergeIfStmt` 之后运行。它在每个最终物理 `dist_put_` 前递增
sender generation，并将 `dist_expect_` 改写为 receiver expected 递增。第一版仍要求
用户显式 sender wait，后续自动 wait 也归该 pass 负责。

这些 pass 不改变非多 Rank kernel。现有 `InjectSunmmioSync` 只增加由 dist leaf 检测
门控的本地兼容处理：`dist_put_` 读取 source 前等待本地 DMA producer token，且不递归
扫描远端 destination region。它不处理 receiver signal generation、sender completion
或自动 sender wait；这些语义仍归属于 dist op 和 `InjectDistSync`。

正式 pass 顺序确定为：

```Plaintext
InferSramScope
  -> PlanDistSignals
  -> LowerDistRouting
  -> LegalizeSunmmioDataPath
  -> SunmmioLayoutInference
  -> LowerDistCommunication
  -> LowerTileOp
  -> ...
  -> SplitHostDevice
  -> MergeIfStmt
  -> InjectDistSync
  -> InjectSunmmioSync
```

四个 dist pass 都先检测对应层级的 dist op 和 `world_size`。没有相关 op 的函数直接返回；
后三个 pass 对非多 Rank 函数直接返回；`PlanDistSignals` 在单 Rank 且出现 dist op 时负责
给出明确错误。

仍需在实现时确定：

- 后续 endpoint 扩展所需 Rank/core symbol 的解析位置。

- Collective 保留、标准化或展开的策略。

- 后续复杂控制流下每个 pass 的完整 invariant 和错误条件。

本轮只处理 TIR，不实现 leaf op 到 SUVM 的 codegen。

### 3\.8 测试与调试工具

所有新增测试放在 `testing/python/sunmmio/inter_rank/` 下，至少包含：

- 前端语法和 TIR script 测试。

- 每个新增 pass 的处理前后结构测试。

- 最终 device kernel TIR 测试。

- 非法 endpoint、region、signal 和 collective 的负向测试。

- 使用现有 lowering 工具打印指定 pass 和最终 TIR 的调试用例。

## 4\. 本轮开发阶段

### 4\.1 阶段一：基础语法搭建

* [x] 建立 T\.dist 模块和导出路径。

* [x] 建立由普通 Python int world\_size 驱动的内部 Rank 编译上下文。

* [x] 定义编译期 world\_size 属性和 T\.dist\.world\_size\(\)。

* [x] 定义 T\.dist\.RankId 及运行时参数索引标记。

* [x] 为 MeshTensor 增加 rank\_placement 和三级 shape 元数据。

* [x] 增加 Rank/core 两层有效 extent 接口。

* [x] 定义 signal 的前端对象和高层 TIR 表示。

* [x] 注册最基础的 P2P/wait 高层 TIR op。

* [x] 确保 signal 和基础通信语法可以构造、打印并通过基础编译流水线。

当前基础实现已经能够将 `world_size`、RankId 参数索引和 Rank placement 元数据保留到
最终 device kernel TIR。Signal 和基础通信 op 与阶段二最小闭环合并实现，不再把
signal 作为脱离 `put/wait` 的独立交付。

### 4\.2 阶段二：最小闭环

目标语义为：

```Python
# rank_id 是由 kernel 启动方传入的 PrimFunc 运行时 int32 参数。
signal = T.dist.signal()

peer = (rank_id + 1) % world_size
T.dist.put(src, dst, dst_rank=peer, signal=signal)
T.dist.wait_signal(signal, dst=dst)
T.dist.wait()
```

第一版限制：

- 目标是其他 Rank 上相同的 local core。

- BufferRegion 大小静态确定。

- 使用显式 receiver signal wait 和显式 sender wait。

- 暂不进行任意 core 转发或自动同步。

- 最终输出可检查的 device kernel TIR，不进入 codegen。

最小闭环验收：

* [x] DSL 可以稳定生成高层 TIR。

* [x] 语义检查和 lowering pass 可以运行。

* [x] 最终 device TIR 只包含预期的底层分布式操作。

* [x] 可以打印并断言每个关键 pass 的处理结果。

* [x] 本地 DMA producer token 在 dist\_put 前完成等待。

* [x] 同一 signal 可以在串行循环中按 uint8 generation 复用。

### 4\.3 阶段三：扩展通信场景

在最小闭环稳定后，逐项设计和实现：

* [x] 根据 receiver destination scope 自动规划 SRAM/DRAM flagreg。

* [x] 支持用户显式选择 signal kind，DRAM memory signal 不自动降级。

* [x] 支持多 signal 和交错多次 put，expected state 独立推进。

* [x] 禁止同一 signal 混用 SRAM 和 DRAM destination scope。

* [x] 前端支持 dst\_row，对等 route 走 fast path，cross\-row route 先在发送 Rank 内转发。

* [x] Cross\-row 使用显式三字段 routed\_put 和两字段 routed\_peer\_put 中间 TIR。

* [x] Signal expected 根据 logical destination 规划，sender generation 与 receiver expected 分离。

* [ ] 公共 T\.dist\.routed\_put 保持三字段 row route，并增加调用级可选 src\_rank； 省略时为当前 Rank，显式指定时由编译器生成 Rank guard。（0\.17 设计已确认，待实现）

* [x] 外层 Rank/column\-uniform 条件可保留，receiver expectation 会被提升并紧凑汇总。

* [x] 固定 extent 的不同 BufferRegion 和非零/表达式 offset。

* [ ] 支持 RSRAM \-\> RSRAM、RSRAM \-\> DRAM、DRAM \-\> RSRAM 和 DRAM \-\> DRAM 四种 scope 组合。（DRAM \-\> RSRAM 设计已确认，待实现）

* [x] 本轮 frontend endpoint 仅支持 Rank/row，source/destination column 隐式保持当前 column；不开放 column/core 参数。

* [x] 多个连续 routed P2P 使用独立 compiler staging，由显式 sender wait 管理生命周期。

* [x] T\.dist\.signals\(count\)、编译期整数索引和 T\.dist\.wait\_all 顺序展开。

* [x] 显式静态 route 的 endpoint、重复表项和唯一物理 signal sender 检查。

* [x] 本 Rank 同 row route 降级为 copy；cross\-row RSRAM 使用 T\.comm\.put；cross\-row DRAM destination 使用目标 RSRAM staging 和 writeback。

* [ ] SignalKind 扩展为六个完整枚举，TIR kind 改为稳定字符串，并按 inc/value/memory 规则处理容量、sender 和 generation。（0\.18 设计已确认，待实现）

阶段三明确遗留，不阻塞当前静态 P2P 收口：

- column/core endpoint、跨 column、接收侧转发和 proxy。

- 静态有界但运行时变化的 extent、source\-local count/valid 和数据驱动 route；这些进入
all\-to\-allv 或其他动态通信协议。

- `wait_any`、SignalList 动态 TIR expression 索引和 backend 原生批量 wait。

- 自动 sender wait pass、循环内 cross\-row staging 复用和自动 buffer 生命周期分析。

- SRAM/DRAM memory signal 的 cross\-row 地址与 wait 路径。

- 一般 destination region alias/多写者冲突分析；当前由用户负责。

- compiler staging 的资源复用和多个真正并发 P2P 的通道调度优化。

每一种场景都必须先确认语义和最终 TIR，再实现对应代码。

### 4\.4 阶段四：Collective Op

在统一的 P2P 和 signal 模型稳定后，逐项支持：

- `T.dist.all_gather`。

- `T.dist.all_reduce`【暂缓】。

- `T.dist.all_to_all`。

- `T.dist.all_to_allv`。

- 后续确认需要的其他 collective。

Collective 使用独立高层 TIR op，frontend 不直接展开通信协议。统一规划一个
`LowerDistCollectives` pass 作为所有 collective 的 lowering 入口，不按 collective 类型
或静态/动态分别注册多个平行 pass。该 pass 位于 `ResolveSunmmioMeshSymbols` 之后、
`InferSramScope` 之前，使展开产生的 put、routed\-put、copy、临时 buffer 和 signal 继续
复用现有 scope、signal、routing、layout 和 sync pipeline。

Lowering 遵守以下原则：

```Plaintext
当前信息足以生成稳定下层 TIR
  -> 直接 lower 为基础 P2P、signal、wait 和本地计算

仍依赖后续 compiler scope/layout/signal/resource analysis
  -> 先 lower 为语义明确的中间 TIR
  -> 等相关 analysis 写入必要信息
  -> 只有此时才增加后续 pass 继续 lowering
```

动态协议包含多个运行时阶段，不等于需要多个 compiler pass。如果
`LowerDistCollectives` 可以直接生成下游能够处理的 runtime loop、condition 和动态基础
通信 op，则仍在同一个 pass 中完成。只有动态中间 op 必须跨越独立 compiler analysis，
且 analysis 结果是继续 lowering 的必要输入时，才新增例如
`LowerDynamicDistCommunication` 的后续 pass；当前不预先注册空的动态 pass。

第一批静态 collective 的目标输出为：

```Plaintext
T.dist.signal / signals
T.dist.put / routed_put
T.dist.wait_signal / wait_all / wait
T.copy、local reduce 和其他已有本地 TileOp
```

Collective lowering 不直接生成 `dist_put_` 等稳定 device leaf，也不自行分配最终 signal
kind/index。`world_size=1` 表示未启用 Rank 间通信，此时出现任何 collective op 都由
`LowerDistCollectives` 明确报错，不进行本地 copy、identity 或 reduce 降级。本轮仍只
支持 Rank/row endpoint，collective 不引入 column/core 参数或跨 column route。

每个 collective 需要确定：

- 输入输出 shape 和数据排列。

- 参与 Rank/core 范围。

- Signal 数量和完成语义。

- 在当前信息充分时展开为基础 P2P/local TIR；信息不足时需要保留的中间 TIR 合同。

- 静态和动态参数的支持边界。

- `world_size=1` 的负向编译测试。

- 正向、负向和最终 device TIR 测试。

## 5\. TIR 处理检查点

调试工具需要持续支持以下检查点：

1. DSL 初始 TIR。

2. 分布式语义检查后的 TIR。

3. P2P 或 collective lowering 后的 TIR。

4. Host/device split 后的 TIR。

5. 分布式同步处理后的 TIR。

6. `OptimizeForTarget` 完成后的 device kernel TIR。

每个新增 pass 都需要在文档中说明它位于哪个检查点、消费什么信息、产生什么信息。

## 6\. 与 SUVM 的未来对接

本轮不实现 SUVM codegen，但最终 device TIR 需要为未来对接保留清晰边界：

- 底层分布式 leaf op 具有稳定且完整的语义。

- 不在 TIR op 中硬编码尚未确认的 SUVM 临时 API 名称。

- 与硬件相关的地址、signal storage、fence 和 descriptor 细节暂不展开。

- 后续获得 SUVM 参考实现后，再补充 TIR 到 SUVM 的映射设计。

- 如果 SUVM 能力限制影响当前 TIR 契约，再回到对应模块讨论和修订。

## 7\. 逐块设计顺序

后续按照以下顺序逐块讨论并更新本文档：

1. Rank、world 和 endpoint 的执行模型。

2. Signal 的前端对象与 TIR 表示。

3. `T.dist.put` 的前端和高层 TIR。

4. `T.dist.wait_signal` 及完成语义。

5. 最小闭环所需的 lowering pass 和最终 device TIR。

6. 多 endpoint、region 和 signal 场景。

7. 各类 collective op。

8. 整体 TIR 契约整理和本轮验收。

## 8\. 已确认事项

|日期|事项|状态|
|---|---|---|
|2026\-08\-26|本轮开发止于最终 device kernel TIR|已确认|
|2026\-08\-26|SUVM codegen 及后续流程不在本轮范围内|已确认|
|2026\-08\-26|使用独立的 `T.dist` 命名空间|已确认|
|2026\-08\-26|第一版 `world_size` 是同名普通 Python `int` 外层编译参数|已确认|
|2026\-08\-26|现有 JIT 自动将 `world_size` 纳入缓存 key|已确认|
|2026\-08\-26|`world_size` 通过内部编译上下文供 MeshTensor 和 DSL 使用|已确认|
|2026\-08\-26|`T.dist.world_size()` 第一版直接返回编译期 `IntImm`|已确认|
|2026\-08\-26|`world_size` 缺省为 1，表示不启用 Rank 间通信|已确认|
|2026\-08\-26|单 Rank kernel 出现 dist 通信 op 时编译报错|已确认|
|2026\-08\-26|`rank_id` 是带语义标记的运行时 `int32` PrimFunc 参数|已确认|
|2026\-08\-26|`rank_id` 第一版使用 `T.dist.RankId` 专用参数注解|已确认|
|2026\-08\-26|`T.Kernel()` 只表示当前 Rank 内的 core mesh|已确认|
|2026\-08\-26|MeshTensor 增加独立于 core placement 的 `rank_placement`|已确认|
|2026\-08\-26|MeshTensor 使用 global、rank、local 三级 shape|已确认|
|2026\-08\-26|省略 `rank_placement` 表示每个 Rank 具有完整 shape|已确认|
|2026\-08\-26|Signal 语法与阶段二最小闭环合并实现|已确认|
|2026\-08\-26|Signal API 使用 `T.dist.signal()` 和 `T.dist.signals()`|已确认|
|2026\-08\-26|Signal 类型统一由 `T.dist.SignalKind` 表达，不再单独传 scope|已确认|
|2026\-08\-26|SignalKind 初始包含 SRAM\_FLAGREG、DRAM\_FLAGREG、DRAM\_MEMORY|初始方案，已由 0\.18 六类设计替代|
|2026\-09\-01|SignalKind 调整为六个完整枚举；显式选择不改写，省略时推断完整 kind|已确认|
|2026\-09\-01|TIR resolved kind 使用 `sram_flagreg_inc` 等稳定字符串，不再使用数字编码|已确认|
|2026\-09\-01|Signal scope 必须等于 receiver destination scope|已确认|
|2026\-09\-01|自增 flagreg 允许多 sender；value flagreg/memory 要求每 sender 独立 signal|已确认|
|2026\-08\-27|Frontend 只分配 logical id，`PlanDistSignals` 按 kind 分配最终 index|已确认|
|2026\-08\-27|Signal kind 缺省为自动，由 receiver destination scope 推断|已确认|
|2026\-08\-27|DRAM memory signal 只能显式选择，flagreg 超量不自动降级|已确认|
|2026\-08\-27|同一 signal 的所有 put/wait destination scope 必须一致|已确认|
|2026\-08\-26|`T.dist.signal` 生成专用 TIR op 和 signal handle|已确认|
|2026\-08\-28|Frontend signal op 使用 `dist_signal_decl`，规划后改为 `dist_signal`，两者参数语义不复用|已确认|
|2026\-08\-26|`dist_put/dist_wait_signal` 注册为标准 TileOperator|已确认|
|2026\-08\-27|Dist primitive 由独立 `LowerDistCommunication` 生成稳定 leaf，原有 `LowerTileOp` 不增加 dist 特判|已确认|
|2026\-08\-26|`wait_signal` 第一版强制要求 `dst`|已确认|
|2026\-08\-26|Expected generation 在最终 TIR 中显式表示|已确认|
|2026\-08\-27|Expected generation 由 `LowerDistCommunication` 隐式分配并写入稳定 leaf 参数|已确认|
|2026\-08\-27|Generation 由 put 推进，wait 只读取且不消费|已确认|
|2026\-08\-27|Generation state 的递增在 pipeline 处理后由 `InjectDistSync` 插入|已确认|
|2026\-08\-27|Put leaf 携带 sender generation，wait leaf 携带 receiver expected，两者独立维护|已确认|
|2026\-09\-01|本轮 frontend endpoint 只支持 Rank/row，column 隐式保持当前 column，不开放 column/core 参数|已确认|
|2026\-08\-27|Cross\-row 物理路径固定为发送 Rank 内先转发，再执行对等 Rank put|已确认|
|2026\-08\-28|全部 route 对等时不生成 routed put，只有 cross\-row 生成显式 route table|已确认|
|2026\-08\-28|普通 route 为 `[origin_src_row, dst_rank, dst_row]`，peer route 为 `[peer_row, dst_rank]`|已确认|
|2026\-08\-28|`routed_put` 不执行 Rank 发送，仅传递 signal；`routed_peer_put` 才表示实际对等发送|已确认|
|2026\-08\-28|Signal expected 根据 peer route table 紧凑汇总，不在 TIR 中展开 Rank 路由|已确认|
|2026\-09\-01|公共 `T.dist.routed_put` 的 route 保持三字段，并增加调用级 `src_rank=None`；显式 Rank 降为外层 Rank guard|已确认|
|2026\-08\-28|Rank 程序使用外层 `rank_id` 条件，column 条件也可外置；row 选择必须进入 route table|已确认|
|2026\-08\-28|Source Rank/column 条件保留物理发送，receiver expectation 提升到 source 条件外|已确认|
|2026\-08\-28|`T.dist.signals` 第一版只支持编译期整数 count 和 Python 整数索引|已确认|
|2026\-08\-28|`T.dist.wait_all` 展开为多个单 signal wait，不增加最终 device leaf|已确认|
|2026\-09\-01|四种 RSRAM/DRAM source\-destination scope 组合全部支持|已确认|
|2026\-08\-28|DRAM 是每个 compute core 独有的 core\-local 地址空间，TIR `global` 不表示 Rank 共享|已确认|
|2026\-08\-28|本 Rank 同 row 使用 copy；cross\-row DRAM 经目标 RSRAM staging 写回；本地路径不推进 dist signal|已确认|
|2026\-08\-28|用户显式管理 source/staging 生命周期，compiler forwarding staging 按 routed put 独立分配|已确认|
|2026\-08\-28|静态 route 检查 endpoint、重复表项和同一 signal 的唯一物理 sender|初始方案，0\.18 改为按更新机制区分|
|2026\-08\-28|正式 dist pipeline 收敛为 4 个 pass，每个 pass 使用独立 `*dist*.cc` 文件并在内部组织多阶段|已确认|
|2026\-09\-01|SRAM/DRAM memory signal 的 cross\-row 路径、动态 route、循环内 staging 复用留待后续|已确认|
|2026\-08\-26|第一版显式书写 sender `T.dist.wait()`|已确认|
|2026\-08\-26|后续允许编译器自动插入并隐藏 sender wait|已确认方向|
|2026\-08\-26|Dist 处理使用可自动跳过的独立 pass 框架|已确认|
|2026\-08\-26|先完成基础语法和最小闭环|已确认|
|2026\-08\-26|最小闭环后扩展多种通信场景|已确认|
|2026\-08\-26|本轮最终继续支持多种 collective op|已确认|
|2026\-09\-01|Collective 统一由 `LowerDistCollectives` 处理；能完整 lower 时直接生成基础通信，否则保留中间 TIR|已确认|
|2026\-09\-01|动态协议不自动对应第二个 pass；仅当中间 TIR 必须跨越独立 compiler analysis 时再新增后续 pass|已确认|
|2026\-09\-01|`world_size=1` 时出现任何 collective op 必须编译报错，不降为本地操作|已确认|
|2026\-08\-26|所有新增测试放在 `testing/python/sunmmio/inter_rank/`|已确认|

## 9\. 参考资料

- 参考草稿：
`testing/my_test/ai_docs/design/signal_base_design/sunmmio_multi_rank_communication_core_design.md`

- 现有 Rank 内通信 frontend：`tilelang/language/comm.py`

- 现有 Rank 内通信 TileOp：`src/op/comm.h` 和 `src/op/comm.cc`

- 现有 SunMMIO 同步 pass：`src/transform/inject_sunmmio_sync.cc`

- Rank 间 TIR 检查工具与测试：`testing/python/sunmmio/inter_rank/`

