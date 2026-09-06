# sunmmio\_inter\_rank\_pass\_design

# SunMMIO Rank 间通信 Pass 设计与现状

## 1\. 文档范围

- 状态：阶段三实现现状与 0\.19 已确认调整

- 版本：0\.4

- 最后更新：2026\-09\-01

- 范围：TileLang DSL 生成高层 TIR 后，到最终 SunMMIO device TIR 为止

本文档单独记录 Rank 间通信相关 pass 的当前安排、主要功能、输入输出和顺序依赖。
SUVM codegen、运行时地址注册和真实硬件指令不在本文档范围内。

## 2\. 当前 Pipeline

SunMMIO 生产 pipeline 中与 Rank 间通信直接相关的顺序为：

```Plaintext
ResolveSunmmioMeshSymbols
  -> InferSramScope
  -> PlanDistSignals                 [新增 dist pass]
  -> LowerDistRouting                [新增 dist pass]
  -> LegalizeSunmmioDataPath
  -> SunmmioLayoutInference
  -> LegalizeSunmmioGemm
  -> LowerDistCommunication          [新增 dist pass]
  -> LowerTileOp
  -> ...
  -> HoistBlockAnnotationsToFuncAttrs
  -> AnnotateDeviceRegions
  -> SplitHostDevice
  -> MergeIfStmt
  -> InjectDistSync                  [新增 dist pass]
  -> InjectSunmmioSync
  -> MakePackedAPI
```

四个新增 dist pass 分别对应四个必须分开的 TIR 阶段：

```Plaintext
逻辑 signal declaration
  -> 已规划 signal 与高层通信
  -> 已确定的物理 route
  -> 稳定 device leaf
  -> pipeline 后的 generation/expected 更新
```

Collective 实现后，计划在 `ResolveSunmmioMeshSymbols` 与 `InferSramScope` 之间插入统一的
`LowerDistCollectives`。当前生产 pipeline 尚未注册该 pass。

## 3\. 新增 Dist Pass

### 3\.1 `PlanDistSignals`

- 文件：`src/transform/plan_dist_signals.cc`

- 阶段：`LowerAndLegalize`

- 位置：`InferSramScope` 之后，`LowerDistRouting` 之前

输入主要包含：

```Plaintext
dist_signal_decl(requested_kind, logical_id)
tileop.dist_put
dist_routed_put
tileop.dist_wait_signal
dist_wait_all
dist_wait_send
```

内部当前分为两个阶段：

1. Signal resource planning

    - 收集 signal declaration 及其全部 put/wait destination。

    - 目标设计支持六个完整 `SignalKind` 枚举；显式 kind 保持不变，省略时推断完整 kind。

    - Signal scope 必须与 receiver destination scope 完全一致。

    - TIR kind 目标表示为 `sram_flagreg_inc` 等稳定 `StringImm`，不再使用数字编码。

    - 分配每种 kind 内的 signal index。

    - 将 `dist_signal_decl(requested_kind, logical_id)` 改写为
    `dist_signal(kind, index)`。

    - 目标 `tl.dist.signal_counts` 为六个稳定字符串到数量的具名 Map；旧三元素数组和兼容
    `tl.dist.sram_signal_count` 待移除。

2. High\-level communication validation

    - 检查 `world_size=1` 时出现 dist 通信 op 的错误。

    - 检查 signal kind/index；SRAM/DRAM 自增 flagreg 各 8 个，value flagreg 各 32 个，
    memory 使用 logical slot。

    - 检查 source/destination scope、dtype、静态 BufferRegion 和元素数量。

    - 检查 put、wait、wait\-all 和 routed\-put 的基础参数合同。

    - 调整后的合同支持 `RSRAM -> RSRAM`、`RSRAM -> DRAM`、`DRAM -> RSRAM` 和
    `DRAM -> DRAM` 四种传输 scope 组合。Signal kind 仍只由 receiver scope 决定。
    当前实现中的 `DRAM -> RSRAM` 拒绝检查待统一移除。

输出约束：

- 不再存在 `dist_signal_decl`。

- 被使用的 signal 均表示为 `dist_signal(kind, index)`。

- 高层 put、wait 和 route 尚未展开。

跳过条件：没有任何高层 dist op 时原样返回。单 Rank 且出现 dist op 时不跳过，而是报错。

### 3\.2 `LowerDistRouting`

- 文件：`src/transform/lower_dist_routing.cc`

- 阶段：`LowerAndLegalize`

- 位置：`PlanDistSignals` 之后，`LegalizeSunmmioDataPath` 之前

输入主要包含已规划 signal，以及逻辑 `tileop.dist_put` 或公共 `dist_routed_put`。

内部当前分为三个阶段：

1. Local Rank lowering

    - 编译期枚举 Rank/row，判断 route 是否满足 `dst_rank == rank_id`。

    - 本 Rank 同 row route 改写为 copy。

    - 本 Rank cross\-row RSRAM destination 改写为 `T.comm.put`。

    - 本 Rank cross\-row DRAM destination 先写入目标 core 的 RSRAM staging，再由目标
    core copy 到自己的 DRAM。

    - 本 Rank route 不生成 Rank 间 signal expectation。

2. Static route planning

    - 检查 destination Rank/row 范围和静态可解析性。

    - 拒绝依赖 BufferLoad 的动态 route。

    - 检查外层条件是否对同一 Rank/current column 的所有 row 一致。

    - 对等 route 改写为 `tileop.dist_peer_put`。

    - cross\-row route 先形成三字段 `[src_row, dst_rank, dst_row]` route table。

    - 检查公共 routed\-put 的重复 route。

3. Physical route lowering

    - 消费三字段 `dist_routed_put`。

    - 对 cross\-row RSRAM/DRAM source 创建独立 RSRAM staging。

    - 生成发送 Rank 内的 `T.comm.put`。

    - 生成两字段 `[peer_row, dst_rank]` route table 和
    `tileop.dist_routed_peer_put`。

    - 当前拒绝循环内 cross\-row staging 和旧 `DRAM_MEMORY + cross-row`；0\.18 的 SRAM/DRAM
    memory cross\-row 地址与 wait 路径仍留待后续。

输出约束：

- 不再存在逻辑 `tileop.dist_put` 或三字段 `dist_routed_put`。

- Rank 间发送只表示为 `tileop.dist_peer_put` 或
`tileop.dist_routed_peer_put`。

- signal handle 仍然存在，但 expected/generation state 尚未创建。

跳过条件：非多 Rank 函数或没有相关高层 dist op 时原样返回。

### 3\.3 `LowerDistCommunication`

- 文件：`src/transform/lower_dist_communication.cc`

- 阶段：`LowerAndLegalize`

- 位置：`SunmmioLayoutInference` 和其他高层 legalization 之后，`LowerTileOp` 之前

输入主要包含：

```Plaintext
dist_signal(kind, index)
tileop.dist_peer_put
tileop.dist_routed_peer_put
tileop.dist_wait_signal
dist_wait_all
dist_wait_send
```

内部当前分为两个阶段：

1. Receiver expectation planning

    - 根据 direct/routed peer route 枚举 source Rank 和物理 peer row。

    - 按 `(dst_rank, peer_row, signal)` 汇总 receiver expected delta。

    - 将 Rank/column 条件换算为 receiver endpoint 上的紧凑 `Select` 表达式。

    - `flagreg_inc` 允许同一 receiver endpoint/signal 聚合多个物理 sender。

    - `flagreg_value/memory` 要求同一 receiver endpoint/signal 只有一个物理 sender。

    - 生成内部 `dist_expect(signal, delta)` marker。

2. Stable leaf lowering

    - 为每个被使用的 signal 创建 sender generation 和 receiver expected state；state dtype
    由 resolved kind 决定，不再统一固定为 `uint8`。

    - 将 direct/routed peer put 改写为稳定 `dist_put_`。

    - 将 `wait_signal` 改写为稳定 `dist_wait_signal_`。

    - 将 `wait_all` 按 signal 顺序展开为多个 `dist_wait_signal_`。

    - 将 `dist_expect` 改写为 `dist_expect_`。

    - 消除 `dist_signal` 和 signal handle。

输出约束：

- 不再存在任何高层 dist TileOperator、route table 或 signal handle。

- `dist_put_` 和 `dist_wait_signal_` 的参数数量及含义从此保持稳定。

- `dist_expect_` 保留到后续 `InjectDistSync`。

该 pass 必须晚于 `SunmmioLayoutInference`，因为 peer put、routed peer put、comm put 和
staging 需要以高层 TileOperator 形式参与 layout inference；它必须早于 `LowerTileOp`，
因为 dist peer TileOperator 不提供普通的 `LowerTileOp` 降级路径。

跳过条件：非多 Rank 函数或没有相关高层 dist op 时原样返回。

### 3\.4 `InjectDistSync`

- 文件：`src/transform/inject_dist_sync.cc`

- 阶段：`OptimizeForTarget`

- 位置：pipeline 规划、host/device split 和 `MergeIfStmt` 之后，`InjectSunmmioSync` 之前

当前功能：

- 在每个最终物理 `dist_put_` 前按 resolved kind 推进 sender generation。

- 将 `dist_expect_(expected, delta)` 按 resolved kind 改写为 receiver expected 更新。

- 稳定 put leaf 统一保留 `send_generation`；自增 flagreg 后端忽略，value/memory 使用。

- 不修改 `dist_put_`、`dist_wait_signal_` 的稳定参数合同。

- wait 只读取 expected，不推进 expected。

- 当前不自动插入 sender wait，用户仍显式书写 `T.dist.wait()`。

输出约束：不再存在 `dist_expect_`，保留 `dist_put_`、`dist_wait_signal_` 和
`dist_wait_send` 供未来 codegen 使用。

跳过条件：非多 Rank 函数或没有稳定 dist leaf 时原样返回。

## 4\. 修改的既有 Pass

|Pass|代码位置|当前 Rank 间通信相关改动|
|---|---|---|
|`HoistBlockAnnotationsToFuncAttrs`|`src/transform/hoist_block_annotations_to_func_attrs.cc`|将 `tl.dist.world_size`、`tl.dist.signal_counts` 和兼容 signal count 属性加入 device function 属性传播集合。|
|`SplitHostDevice`|`src/transform/split_host_device.cc`|保证运行时 `rank_id` 进入 device kernel；参数重排后重新生成 `tl.dist.rank_id_param_index`；host function 删除失效索引。|
|`MakePackedAPI`|`src/transform/make_packed_api.cc`|MeshTensor 参数检查优先使用单 Rank 的 `rank_shape/rank_strides`，缺失时回退到原有 global metadata。|
|`InjectSunmmioSync`|`src/transform/inject_sunmmio_sync.cc`|将 `dist_put_` 作为本地 source 的同步读取，将 `dist_wait_signal_` 作为 local destination 可见点，并让 `dist_wait_send` 参与既有同步语句排序；不处理 generation/expected。|

## 5\. 复用但未为 Dist 修改的关键 Pass

|Pass|当前作用|
|---|---|
|`ResolveSunmmioMeshSymbols`|在 dist pass 前将 `mesh_nrows/mesh_ncols` 解析为目标硬件常量。|
|`InferSramScope`|在 signal planning 前确定 RSRAM scope，使 signal kind 能依据 receiver scope 推断。|
|`LegalizeSunmmioDataPath`|处理 `LowerDistRouting` 生成的本地 copy/comm 数据路径和必要 staging。|
|`SunmmioLayoutInference`|为 comm put、dist peer put、routed peer put 和相关 staging 推断 layout。|
|`LegalizeSunmmioGemm`|继续处理 GEMM 高层合法化；与 dist expectation 没有直接数据依赖。|
|`LowerTileOp`|降级剩余 TileOperator；dist peer TileOperator 在进入它之前已经被消除。|
|`MergeIfStmt`|在 `InjectDistSync` 前完成既有条件结构合并，使 generation 更新对应最终物理发送顺序。|

## 6\. 当前职责分布

|事项|当前负责位置|
|---|---|
|world size、RankId、rank placement、逻辑 signal ID|Python frontend/Builder/MeshTensor|
|六类 signal kind 推断、scope 绑定与 index 分配|`PlanDistSignals`|
|高层通信的 scope/dtype/static region 基础检查|`PlanDistSignals` 内部 validation|
|endpoint、条件和 route 合法性|`LowerDistRouting` 内部 route planning|
|本 Rank copy/comm 降级|`LowerDistRouting` 内部 local Rank lowering|
|远端 cross\-row staging 和 peer route|`LowerDistRouting` 内部 physical route lowering|
|receiver expected delta 与按 kind 区分的 sender 检查|`LowerDistCommunication` 内部 expectation planning|
|signal state 和稳定 leaf 生成|`LowerDistCommunication` 内部 leaf lowering|
|generation/expected 的最终更新位置|`InjectDistSync`|
|dist leaf 与本地 DMA/token 数据依赖|`InjectSunmmioSync`|
|Rank 元数据跨 host/device 保留|Hoist/Split/MakePackedAPI 相关既有 pass|

## 7\. 整体 Pass 规划

整体 pass 按 signal 资源、通信路由、通信 leaf 和最终同步四个阶段逐项确认。本节只记录
已经确认的职责。

### 7\.1 `PlanDistSignals`

定位：独立的 signal 资源规划 pass，位于 `InferSramScope` 之后、`LowerDistRouting` 之前。

输入与输出：

```Plaintext
dist_signal_decl(requested_kind, logical_id)
  -> PlanDistSignals
dist_signal(kind, index)
```

负责：

- 检查 requested kind、logical ID、signal 引用关系和 flagreg 容量。

- 收集普通 put、routed put、wait、wait\-all 以及后续 collective 对 signal 的使用。

- 汇总 receiver destination scope，检查同一 signal 的 scope 一致性。

- 根据 destination scope 和用户显式 kind 确定最终 signal kind。

- 为每种 kind 分配 compiler\-owned index，用户不指定 index。

- 写入 `tl.dist.signal_counts`，并保证输出中不再存在 `dist_signal_decl`。

- 作为第一个 dist pass，检查单 Rank kernel 中出现 dist 通信 op 的错误。

同一个 signal 可以被多个通信操作使用。`PlanDistSignals` 只规划共享的 signal 资源，不在
此处限制物理 sender 数量，也不计算 expected delta。多个入边是否能聚合到一个 signal，
由后续 completion planning 根据 signal kind 和通信语义处理。

不负责：

- payload source/destination 的 dtype、元素数量和传输 scope 组合。

- endpoint、route、外层条件和 staging 合法性。

- sender 数量、expected delta、generation 更新和 sender wait。

当前需要调整：

- 将 `ValidateStaticTransfer`、payload dtype/元素数量、put/routed\-put region 合同等通用通信
检查从 `PlanDistSignals` 移入 `LowerDistRouting` 的内部 validation 阶段。

- `PlanDistSignals` 保留 destination scope 收集、kind/index 规划及所有 signal 专属检查。

- 后续新增 collective op 时，将其 signal/destination 使用注册到统一 signal use collector。

### 7\.2 `LowerDistRouting`

定位：核心通信路由 pass，负责校验 signal 规划完成后的高层 dist 通信，并将逻辑 source/
destination endpoint 转换为本 Rank 数据路径或真实 Rank 间物理 route。

位置：`PlanDistSignals` 之后、`LegalizeSunmmioDataPath` 之前。

Source endpoint 语义：

```Plaintext
src_rank=None -> 当前 rank_id
src_row=None  -> 当前 current_row
dst_row=None  -> source row，即对端 row
```

Column/core 边界：

- 本轮 frontend 不接收 `src_column`、`dst_column`、`src_core` 或 `dst_core`。

- Source/destination column 始终隐式使用 `current_col`；显式 route 只改变 row。

- 内部物理 core 由 `row * mesh_ncols + current_col` 构造，不进行 column 路由规划。

- 外层 `current_col` 条件只作为 SPMD predicate 保留，不表示通信 endpoint 携带 column。

- 跨 column 和完整 core endpoint 留待后续设计，不进入本轮 pass 合同。

显式 `src_rank/src_row` 表示哪一个 SPMD endpoint 执行通信，不表示当前 Rank 可以读取其他
Rank 的 source。归一化后的逻辑 route 完整包含：

```Plaintext
[src_rank, src_row, dst_rank, dst_row]
```

负责：

1. High\-level dist validation

    - 确认输入 signal 已表示为规划后的 `dist_signal(kind, index)`。

    - 检查除 signal 资源外的所有高层 dist op 参数合同。

    - 检查 payload source/destination dtype、元素数量、静态 BufferRegion 和传输 scope 组合。

    - 调整后四种 RSRAM/DRAM scope 组合均合法；不因 source 为 DRAM、destination 为
    RSRAM 拒绝。当前实现待同步。

    - 检查 source/destination Rank、row 的类型、范围和静态可解析性。

    - 检查 routed\-put route table、重复项、外层条件及动态 BufferLoad 依赖。

    - 检查 wait、wait\-all 和 sender wait 的基础参数结构。

2. Source endpoint activation

    - `src_rank` 为当前 Rank 时继续处理 route。

    - 显式 `src_rank` 转换为 `rank_id == src_rank` 的 source\-side Rank guard。

    - 显式 `src_row` 进入 route source\-row 选择，不作为远端寻址参数。

    - 不因 `world_size` 自动复制 device TIR；只有用户或上层规划显式提供的 route 才产生
    对应 Rank 条件。

3. Local Rank lowering

```Plaintext
rank_id == src_rank && dst_rank == src_rank
  同 row     -> copy
  cross-row  -> Rank 内 T.comm.put
```

- Cross\-row DRAM destination 使用目标 core 的 RSRAM staging 和 writeback。

- 本 Rank route 不生成物理 Rank put，也不计入 dist signal expected。

4. Remote Rank routing

```Plaintext
rank_id == src_rank && dst_rank != src_rank
  dst_row == src_row  -> direct dist_peer_put
  dst_row != src_row  -> 本 Rank 转发后 dist_routed_peer_put
```

- 非对端通信为 core\-local source 创建 compiler\-owned RSRAM staging。

- 先生成 origin row 到 egress row 的 `T.comm.put`。

- 再由 egress row 对远端相同 row 执行 Rank 间 put。

- RSRAM 和 DRAM 都是 core\-local，cross\-row source 都经过 egress staging。

- 所有 egress 和 destination core 都保持 `current_col`，本轮不产生跨 column route。

输出约束：

- 不再存在逻辑 `tileop.dist_put` 或普通 `dist_routed_put`。

- 本 Rank 路径只保留 copy、`tileop.comm_put` 和必要 staging。

- Rank 间路径只保留 `tileop.dist_peer_put`、`tileop.dist_routed_peer_put` 和 peer route。

- `src_rank/src_row` 已被消费为 Rank guard、显式 core route 或本地操作，不进入最终
`dist_put_` leaf。

- signal handle 仍存在；expected、generation 和 sender wait 不在此 pass 处理。

当前需要调整：

- 增加独立且先于所有 rewrite 的 `DistRoutingValidator` 阶段。

- 将通用 payload/region/传输 scope validation 从 `PlanDistSignals` 移入该 validator。

- 扩展前端和高层 `dist_put`，支持可选 `src_rank/src_row` 并归一化默认当前 endpoint。

- 将内部逻辑 route 统一为 `[src_rank, src_row, dst_rank, dst_row]`。

- 显式 source Rank 在 routing 中转换为外层 Rank guard；物理 peer op 和稳定 leaf 不增加
source Rank 参数。

- 本 Rank 判断从单纯比较 `dst_rank == rank_id` 统一为先激活 source Rank，再比较
`dst_rank == src_rank`。

- 保证 source Rank guard 保留到后续 expectation planning，使 receiver expected 只统计
实际执行的远端发送。

- 更新普通 put、显式 source endpoint、本 Rank、peer 和 cross\-row 的 pass 前后 TIR 测试。

## 8\. Pass 分工待讨论项

以下内容记录当前实现中的职责交界，不表示已经决定修改：

1. Receiver expectation 完全由最终物理 route 推导，但当前位于
`LowerDistCommunication`；需要确认它应继续与 leaf lowering 合并，还是归入 routing。

2. `InjectDistSync` 当前只处理 generation/expected，设计上还承接未来自动 sender wait；
需要确认未来功能加入后是否仍保持单一职责。

3. Rank 元数据分别经过 Hoist、Split 和 MakePackedAPI 三个既有 pass；当前修改遵循各 pass
原有职责，需要持续确认新增 metadata 是否都能沿同一路径传播。

## 9\. 0\.18 Signal 调整后续工作

以下工作均为设计已确认后的实现任务，当前代码尚未完成：

1. Frontend 与初始 TIR

    - 将 `T.dist.SignalKind` 扩展为六个完整枚举：SRAM/DRAM 的 INC flagreg、VALUE
    flagreg 和 memory。

    - `kind=None` 继续生成 `dist_signal_decl("auto", logical_id)`；显式枚举转换为稳定
    `StringImm`，不再写数字。

    - 更新单 signal、SignalList、类型检查、错误信息和 frontend TIR 测试。

2. 统一 kind 元数据

    - 在 C\+\+ 中建立六类 resolved kind 的集中描述表，记录 scope、更新模式、容量、state
    dtype 和多 sender 能力，避免各 pass 分散判断字符串。

    - resolved TIR 名称固定为 `sram_flagreg_inc`、`dram_flagreg_inc`、
    `sram_flagreg_value`、`dram_flagreg_value`、`sram_memory`、`dram_memory`。

    - 将 `tl.dist.signal_counts` 从三元素数组改为六个稳定名称到数量的具名 Map，并移除旧
    `tl.dist.sram_signal_count` 兼容属性及其传播逻辑。

3. 自动规划时序

    - 显式 kind 可在早期完成 scope/capacity 检查，但 `kind=None` 的完整推断可能依赖最终
    sender 拓扑。

    - 确认采用“早期绑定 scope、routing 后分配 kind/index”，还是将完整 signal resource
    planning 移到 route 形成之后。

    - 明确自动 kind 的选择优先级和容量耗尽行为；禁止在多 sender 语义下静默降级为
    VALUE/MEMORY。

4. Expectation 与 sender 规则

    - `flagreg_inc` 对同一 receiver signal 汇总所有物理 sender 的 contribution。

    - `flagreg_value/memory` 检查每个 receiver signal 只有一个物理 sender；同一 sender 的
    连续 put 继续推进 generation。

    - 更新 conditional route、routed put、多 Rank sender 和多次 put 的 expectation 测试。

5. State 与稳定 leaf

    - `LowerDistCommunication` 按 kind 创建对应 dtype 的 generation/expected state。

    - `dist_put_`、`dist_wait_signal_` 的 kind 参数改为 `StringImm`；统一保留
    `send_generation`，INC 后端忽略，VALUE/MEMORY 使用。

    - `InjectDistSync` 移除固定 `uint8` 假设，按 kind/dtype 推进 generation 和 expected。

    - 依据底层接口确认 VALUE flagreg、SRAM/DRAM memory 的写入位宽、初始值、回绕和 wait
    比较语义。

6. 测试与文档

    - 为六个显式 kind 分别增加 frontend、planning、capacity、scope 和最终 leaf 正向测试。

    - 增加 INC 多 sender 正向测试，以及 VALUE/MEMORY 多 sender 负向测试。

    - 更新关键 pass 打印 kernel，使 `dist_signal_decl("auto", ...)` 和 resolved 字符串 kind
    可直接检查。

    - 在底层接口明确后同步更新 low\-level overview 和未来 SUVM leaf 映射文档。

## 10\. Collective Pass 规划

### 10\.1 `LowerDistCollectives`

- 计划文件：`src/transform/lower_dist_collectives.cc`

- 阶段：`LowerAndLegalize`

- 计划位置：`ResolveSunmmioMeshSymbols` 之后、`InferSramScope` 之前

- 当前状态：设计确认，尚未实现或注册

输入为 frontend 保留的高层 collective op，例如：

```Plaintext
tileop.dist_all_gather
tileop.dist_all_reduce
tileop.dist_all_to_all
tileop.dist_all_to_allv
```

该 pass 作为所有 collective 的统一入口，不为每个 collective 或静态/动态场景注册平行
pass。内部按以下阶段组织：

1. Validate

    - 检查参与 Rank、shape、axis、dtype、静态/动态参数和本轮 Rank/row endpoint 边界。

    - 没有 collective op 时原样返回。

    - 存在 collective op 且 `world_size=1` 时明确报错，不生成本地 copy、identity 或
    local reduce。

    - 只有存在 collective op 且 `world_size>1` 时才继续协议选择和 lowering。

2. Normalize and select protocol

    - 规范化输入输出数据排列、参与集合和 completion 语义。

    - 根据 collective 类型、world size 和静态信息选择 ring、direct、reduce\-scatter 等协议。

3. Build logical schedule

    - 生成 logical signal declaration、source/destination route、receiver wait 和 sender wait。

    - 不分配最终 signal kind/index，不直接生成 `dist_put_` 等稳定 leaf。

4. Rewrite

    - 信息充分的协议直接展开为 `put/routed_put/signal/wait`、runtime loop/condition 和已有
    local TileOp。

    - 展开后的 buffer 和通信 op 继续进入 `InferSramScope`、`PlanDistSignals`、
    `LowerDistRouting`、`SunmmioLayoutInference` 和 `LowerDistCommunication`。

输出 invariant：

- 已完整 lower 的 collective 不再保留高层 collective op。

- 静态 collective 输出只使用现有基础 P2P、signal、wait、copy/reduce 等合同。

- 无 collective op 时原样返回。

- 单 Rank kernel 中出现 collective op 时在本 pass 报错，不能按“非多 Rank”直接跳过。

- 所有 route 只使用 Rank/row endpoint，column 隐式保持 `current_col`。

### 10\.2 是否增加后续 Dynamic Pass

动态 collective 有多个运行时阶段，并不自动要求第二个 compiler pass。判断标准是第一个
pass 是否已经能够生成下游理解的稳定 TIR：

```Plaintext
能够直接生成 runtime loop、condition 和稳定 dynamic communication op
  -> 继续由 LowerDistCollectives 一次完成，不新增 pass

只能生成尚未解析的 dynamic intermediate op，且它必须先经过其他 compiler analysis
  -> 保留中间 TIR
  -> analysis 完成后新增后续 pass 继续 lowering
```

这里的“必须跨越独立 analysis”是指继续 lowering 依赖其他 pass 产生的信息，例如：

- `InferSramScope` 确定 metadata/staging scope。

- `PlanDistSignals` 确定 signal kind/index 和多 sender 资源。

- `SunmmioLayoutInference` 确定动态 payload/staging layout。

- 后续明确需要的 channel、metadata slot 或 staging lifetime resource planning。

只有这些 analysis 的输出是 dynamic intermediate op 继续 lowering 的必要输入时，才新增
例如 `LowerDynamicDistCommunication` 的后续 pass。当前不创建、不注册空的动态 pass；
`all_to_allv` 的动态 BufferRegion、metadata/count exchange、零长度 route 和 `wait_any`
合同确定后再作判断。

### 10\.3 初始实现顺序

1. 先注册一个 `LowerDistCollectives`。

2. 依次实现静态 `all_gather`、静态 `all_to_all`；`all_reduce` 按主设计暂缓。

3. 静态 collective 全部展开为现有 P2P 和 local op，并复用现有四个 dist pass。

4. 设计 `all_to_allv` 的动态中间 TIR。

5. 根据动态中间 TIR 是否必须跨越独立 analysis，决定是否增加第二个 pass。

6. 增加 DSL、collective pass 前后、基础 P2P 输出和最终 device TIR 测试。

7. 为每个 collective 增加 `world_size=1` 负向测试，确认不会静默本地化。

