# sunmmio\_inter\_rank\_parallel\_work\_plan

# SunMMIO Rank 间通信并行任务拆分草案

## 1\. 文档状态

- 状态：待确认

- 范围：当前 Rank 间通信下一批并行开发任务

- 目标：在不进入 SUVM/codegen 的前提下，完善 signal、receiver/sender wait，并建立
collective frontend 与 lowering 框架

- 验收终点：各项功能能够从 DSL 降到可检查的 device kernel TIR

本文只负责划分任务和接口所有权。具体语义以
`sunmmio_inter_rank_communication_design.md` 和 `sunmmio_inter_rank_pass_design.md` 为准。

## 2\. 当前共识

### 2\.1 总体边界

1. 本轮实现 frontend 语法、TIR op、分析与 lowering pass，不实现 SUVM、codegen、runtime
ABI 和硬件执行。

2. `world_size` 是编译期正整数，`rank_id` 是 kernel 启动时传入的运行时 `int32` 参数。

3. `world_size=1` 表示未启用 Rank 间通信。此时只要出现 P2P 或 collective dist op，必须
编译报错，不能降级为本地 copy、identity 或 reduce。

4. 当前 endpoint 只支持 Rank 和 row；column 隐式保持 `current_col`。跨 column、完整
`dst_core`、proxy、接收侧转发不属于本批次。

5. Collective lowering 只生成现有高层 P2P、signal、wait 和本地 copy/reduce 等操作，
不直接生成 `dist_put_`、`dist_wait_signal_` 等稳定 leaf。

### 2\.2 Signal 类型和推断

Frontend 保留完整枚举：

```Python
T.dist.SignalKind.SRAM_FLAGREG_INC
T.dist.SignalKind.DRAM_FLAGREG_INC
T.dist.SignalKind.SRAM_FLAGREG_VALUE
T.dist.SignalKind.DRAM_FLAGREG_VALUE
T.dist.SignalKind.SRAM_MEMORY
T.dist.SignalKind.DRAM_MEMORY
```

TIR resolved kind 使用以下稳定字符串，不使用数字：

```Plaintext
sram_flagreg_inc
dram_flagreg_inc
sram_flagreg_value
dram_flagreg_value
sram_memory
dram_memory
```

第一版自动推断范围收紧为：

- `kind=None` 只根据 receiver destination scope 推断对应的 INC flagreg：
`shared.rsram -> sram_flagreg_inc`，`global/DRAM -> dram_flagreg_inc`。

- VALUE flagreg 和 memory signal 必须由用户显式指定。

- 显式 kind 以用户选择为准；编译器只做 scope、容量、sender 和使用规则校验，不替换
kind，也不在容量不足时静默降级。

六种 signal 的资源规则为：

|Resolved kind|容量/endpoint|更新方式|Sender 规则|
|---|---|---|---|
|`sram_flagreg_inc`|8|硬件自动 `+1`|允许多个 sender 共享|
|`dram_flagreg_inc`|8|硬件自动 `+1`|允许多个 sender 共享|
|`sram_flagreg_value`|32|写入明确 generation|每个 sender 使用独立 signal|
|`dram_flagreg_value`|32|写入明确 generation|每个 sender 使用独立 signal|
|`sram_memory`|基本不限|写入明确 generation|每个 sender 使用独立 signal|
|`dram_memory`|基本不限|写入明确 generation|每个 sender 使用独立 signal|

Signal scope 只由 receiver destination scope 决定。同一个 signal 不能混用 SRAM 和 DRAM
destination；显式 kind 与 destination scope 不一致时直接报错。

### 2\.3 State dtype 和 wait 比较合同

根据当前已验证的底层样例，第一版采用：

|Signal 类别|generation/expected dtype|Receiver wait 的“已到达”语义|
|---|---|---|
|SRAM/DRAM INC flagreg|`uint8`|按 8\-bit 回绕比较，waiter 不得落后超过 127 次更新|
|SRAM/DRAM VALUE flagreg|`uint32`（暂定）|`observed >= expected`|
|SRAM/DRAM memory|`uint32`|`observed >= expected`|

这里的比较规则是未来 backend 实现 `dist_wait_signal_` 时的合同。当前 TIR 工作需要正确携带
kind、dtype 和 expected，但不在 TileLang pass 中展开硬件轮询。VALUE flagreg 的位宽缺少
可验证样例，先按 `uint32` 实现并集中封装，后续可随 SUVM 接口调整。

共同的 generation 规则：

- 每次实际发生的物理 peer put 推进一次对应的发送 generation/接收 expectation。

- INC flagreg 可以汇总多个 sender 的 contribution。

- VALUE flagreg 和 memory signal 在同一 receiver endpoint 上只能绑定一个物理 sender；
同一 sender 可以连续多次 put 并推进 generation。

- `wait_signal` 只读取当前 expected，不推进 expected；重复 wait 等待同一个值。

- 本 Rank copy 和 Rank 内 `T.comm.put` 不属于 Rank 间 peer put，不计入 receiver expected。

## 3\. 并行任务总览

|负责人|工作线|核心产出|主要依赖|
|---|---|---|---|
|负责人 A\-杨晓朝|Collective frontend 与 lowering|高层 collective op、`LowerDistCollectives`|复用基础 P2P 合同|
|负责人 B\-赵世杰|Signal 资源规划|六类 kind、推断、容量和 scope 校验|为 D 提供 resolved kind 元数据|
|负责人 C\-肖尧|Sender completion wait|自动 sender wait 分析与插入|在最终调度顺序上分析物理 put|
|负责人 D\-关舟|Receiver wait 与 expected|generation/expected 计算和稳定 wait leaf|消费 B 的 resolved kind 元数据|

四条工作线共享同一基础约束：没有对应 dist op，或不是多 Rank kernel 时，pass 应快速原样
返回；唯一例外是 `world_size=1` 且实际出现 dist op，此时必须明确报错。

## 4\. 负责人 A：Collective Frontend 与 Lowering

### 4\.1 主要工作

1. 增加 collective frontend API 和对应高层 TileOperator/TIR 表达：
`all_gather`、`all_to_all`、`all_reduce`、`all_to_allv`。

2. 为每个 op 明确输入输出 shape、axis、dtype、Rank 参与集合、数据排列和同步语义。

3. 新增并注册独立 `LowerDistCollectives` pass，位置在
`ResolveSunmmioMeshSymbols` 之后、`InferSramScope` 之前。

4. 在 pass 内完成 validation、协议选择、logical schedule 构造和 rewrite。

5. 第一实现顺序为静态 `all_gather`、静态等长 `all_to_all`，再处理 `all_reduce`；
`all_to_allv` 先完成 frontend 和动态协议设计，只有信息足够时才进入 lowering。

6. Lowering 生成 `T.dist.signal/signals`、`put/routed_put`、`wait_signal/wait_all` 及已有
local op，让后续 signal、routing、communication 和 sync pass 继续处理。

### 4\.2 职责边界

- 不分配 signal kind/index；collective 产生的 signal 默认使用 `kind=None`，交给
`PlanDistSignals`。

- 不计算最终 receiver expected，不生成 `dist_expect_`。

- 不直接生成稳定 device leaf。

- 不在本任务中预先增加第二个 dynamic pass。只有动态中间 TIR 必须等待其他独立 analysis
的结果才能继续 lowering 时，再单独提出新增 pass。

- 不实现跨 column、proxy、接收侧转发和 SUVM 映射。

### 4\.3 交付与验收

- 每个已实现 collective 都有 frontend 正向/参数负向测试。

- 测试 `LowerDistCollectives` 前后的 TIR，确认高层 op 被完整消除并只留下基础合同。

- 每个 collective 增加 `world_size=1` 负向测试。

- 在 `test_dist_pass_tir_print.py` 增加少量代表性正向 kernel，打印 collective lowering 后、
signal planning 后和最终 device TIR。

## 5\. 负责人 B：完善 `PlanDistSignals`

### 5\.1 主要工作

1. 将 frontend `SignalKind` 扩展为六类完整枚举，并让初始 TIR 使用稳定字符串 requested
kind；`kind=None` 使用 `"auto"`。

2. 建立集中的 signal kind 元数据，至少包括 scope、更新模式、容量、state dtype、是否允许
多 sender。其他 pass 必须复用该元数据，不能各自维护字符串判断。

3. `PlanDistSignals` 收集普通 put、routed put、wait、wait\-all 以及 collective lowering 后的
signal 使用。

4. 实现第一版自动推断：只推断 scope 对应的 INC flagreg。

5. 实现显式 kind 的 scope 校验、INC/VALUE flagreg 容量校验、logical ID/引用关系校验，
并按 kind 分配 compiler\-owned index。

6. 将 `tl.dist.signal_counts` 改为六种稳定 kind 到数量的具名 Map，并保证跨 host/device
相关 pass 正确传播。

### 5\.2 职责边界

- 不计算 sender generation、receiver expected 或物理 sender contribution。

- 不负责 sender wait。

- VALUE/MEMORY 不参与第一版自动推断；用户未显式选择时不能因 INC 容量不足而自动降级。

- 多 sender 是否合法的最终检查依赖物理 route，由负责人 D 完成；本 pass 只提供 kind 的
`allow_multi_sender` 元数据和可在早期确定的检查。

### 5\.3 交付与验收

- 六种显式 kind 的 frontend、resolved TIR、scope 和容量测试。

- SRAM/DRAM `kind=None` 分别推断到对应 INC flagreg 的测试。

- 显式 kind scope 不匹配、flagreg 超容量、同一 signal 混用 scope 的负向测试。

- 证明无 dist op 时原样返回，单 Rank 出现 dist op 时明确报错。

## 6\. 负责人 C：Sender Completion Wait

### 6\.1 主要工作

1. 参考现有核内异步通信的 buffer access/token 分析，识别每个物理 `dist_put_` 对 source
region 和 compiler staging 的异步占用。

2. 在以下位置自动插入 sender completion wait：
source region 即将被写入或复用之前、compiler staging 即将被覆盖之前、以及仍有未完成
send 的 kernel 退出之前。

3. 保留并识别用户显式 `T.dist.wait()`；显式 wait 清空此前 pending send，禁止重复插入。

4. 正确处理顺序语句、分支、静态循环和 pipeline 处理后的语句顺序。

5. Sender wait 只表示本地 DMA/channel completion，不能被 receiver `wait_signal` 替代。

### 6\.2 建议实现边界

为减少与负责人 D 同时修改 `InjectDistSync` 的冲突，建议将自动 sender wait 实现为独立的
dist 分析/改写阶段，例如 `InsertDistSenderWait`，放在 SunMMIO pipeline 和
`MergeIfStmt` 之后、`InjectDistSync` 之前。它可以复用 `InjectSunmmioSync` 的 region overlap
和控制流分析思路，但不应把大量 dist 专属逻辑继续塞入既有核内通信 pass。

该 pass 是否最终合并回 `InjectDistSync`，等行为和测试稳定后再决定；第一版优先保持文件
所有权和并行开发边界清晰。

### 6\.3 职责边界

- 不处理 receiver signal、expected 或 generation。

- 不改变 `dist_put_` 的参数合同。

- 第一版不做 channel 数量分配和精确 channel 级 wait；在底层 channel 合同明确前，
`T.dist.wait()` 仍表示等待当前 endpoint 此前提交的全部 Rank 间 send。

- 不为普通只读且生命周期未结束的 source 过早插入 wait。

### 6\.4 交付与验收

- source 无复用时直到 kernel 结束才 wait。

- source/staging 被覆盖前自动 wait。

- 连续多个 send 可以合并为一次 wait，不为每个 put 无条件插入。

- 显式 wait 不重复、receiver wait 不清除 sender pending 状态。

- 分支、循环和 cross\-row compiler staging 的针对性测试。

## 7\. 负责人 D：完善 `wait_signal` 与 Expected 计算

### 7\.1 主要工作

1. 在最终物理 route 形成后，按 receiver endpoint 和 signal 汇总实际 peer put。

2. INC flagreg 汇总多个 sender contribution；VALUE/MEMORY 校验每个 receiver signal 只有
一个物理 sender。

3. 为每种 signal 创建正确 dtype 的 sender generation 和 receiver expected state：
INC 使用 `uint8`，VALUE/MEMORY 第一版使用 `uint32`。

4. 在 `InjectDistSync` 中于实际物理 put 前推进对应 sender generation，并在正确的控制流
位置物化 receiver expected 更新。

5. `wait_signal` 和 `wait_all` 使用当前 expected；wait 本身不推进 state，重复 wait 语义
不变。

6. `dist_put_`、`dist_wait_signal_` 使用稳定字符串 kind，并保持稳定参数合同；put 继续携带
`send_generation`，INC backend 可忽略，VALUE/MEMORY 使用。

### 7\.2 比较规则边界

- 当前 pass 只生成正确 kind、dtype 和 expected 参数，不展开 receiver 轮询。

- 未来 backend 对 INC flagreg 使用 8\-bit 回绕“已到达”比较；对 VALUE/MEMORY 使用
`observed >= expected`。

- 动态 active sender、零长度 route 和 `wait_any` 属于 `all_to_allv` 协议设计，不在第一版
expected 完善中提前实现。

### 7\.3 交付与验收

- 单 signal 多次 put、重复 wait、多 signal 交错 put/wait\-all。

- INC 多 sender 聚合的正向测试。

- VALUE/MEMORY 多 sender 共用同一 receiver signal 的负向测试。

- conditional route、routed put、多 Rank sender 的 expected 测试。

- 本 Rank copy/Rank 内转发不增加 expected，只有最终 peer put 增加 expected。

- 检查 INC state 为 `uint8`、VALUE/MEMORY state 为 `uint32`。

## 8\. 共享文件与协作约束

以下文件容易发生并行冲突，需要明确所有权：

|文件/区域|主要负责人|协作规则|
|---|---|---|
|`tilelang/language/dist.py` 中 signal 部分|B|A 的 collective frontend 尽量放独立模块后再导出|
|Collective TileOperator/op 注册文件|A|与现有 P2P op 分文件，避免改动 signal 实现|
|Signal kind 公共元数据|B|D 只消费，不复制定义|
|`plan_dist_signals.cc`|B|A/D 不直接加入特殊分支，统一通过基础 op 合同接入|
|`lower_dist_communication.cc`|D|A/B/C 不在此处实现自己的功能|
|`inject_dist_sync.cc`|D|C 第一版使用独立 sender\-wait 文件|
|`inject_sunmmio_sync.cc`|C（如确有必要）|优先复用 helper，避免改变非 dist 行为|
|`phase.py` 和 transform 导出|集成时串行修改|各负责人先完成 pass 本体，最后按合入顺序统一注册|

所有新 dist pass 遵守：一个 pass 一个带 `dist` 的独立文件；无相关 op 时快速跳过；测试只跑
各自相关集合，不进行日常全量回归。全量回归由项目负责人按阶段统一安排。

## 9\. 依赖和建议合入顺序

并行开发可以同时开始，但建议按以下顺序集成：

1. 先冻结基础 op 参数合同和六类 signal 稳定名称。

2. 合入 B：六类 signal frontend、公共元数据和 `PlanDistSignals`。

3. 合入 D：按 resolved kind 完善 generation/expected 和 receiver wait。

4. 合入 C：在最终调度顺序上自动插入 sender wait。

5. 合入 A：collective frontend 与静态 lowering，并跑穿完整基础 pipeline。

A 可以在 B/D 开发期间只验证 `LowerDistCollectives` 输出的高层 P2P TIR；C 可以基于现有
`dist_put_` 合同独立开发。最终集成测试再覆盖 collective 经过 B、D、C 后的 device TIR。

## 10\. 本批次不处理的事项

- SUVM/codegen、runtime ABI、实际硬件地址和 wait helper 映射。

- 跨 column、显式 source/destination core、proxy、先 put 后转发、接收侧转发。

- 动态 `dst_row`、动态 route table 和动态 signal 索引。

- `wait_any`、动态 active\-count wait 和完整 `all_to_allv` runtime 协议。

- INC 之外的自动 signal kind 推断。

- 精确 PCIe channel 分配和 channel 级 sender wait。

- VALUE flagreg 底层位宽的最终定版；第一版暂用 `uint32`，后续集中调整。

以下已确认但尚未实现的 P2P 补充项不应遗失，是否并入本批次需另行指定负责人：

- `T.dist.routed_put(..., src_rank=...)` 的调用级可选 source Rank。

- `RSRAM -> RSRAM`、`RSRAM -> DRAM`、`DRAM -> RSRAM`、`DRAM -> DRAM` 四种 scope
组合的完整收口。

## 11\. 完成定义

每位负责人交付时至少满足：

1. 功能实现与已确认合同一致，没有引入未确认的后端假设。

2. 新 pass 在无相关 dist op 时不改变 TIR。

3. 仅运行并记录对应模块的针对性测试；不要求日常全量回归。

4. 增加关键 pass 前后 TIR 断言；代表性正向场景补到打印测试。

5. 更新主设计或 pass 设计中受实现影响的章节，删除已经失效的旧描述。

