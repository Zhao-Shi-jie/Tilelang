# Signal资源规划快速指南

> 面向需要理解或扩展SunMMIO Rank间通信Signal系统的开发者

## 📋 文档导航

1. **实现状态报告**: [`person_b_signal_planning_implementation_status.md`](./person_b_signal_planning_implementation_status.md)  
   详细的实现状态、测试覆盖、代码位置和设计细节（本次完成的核心文档）

2. **设计文档**:
   - 通信设计大纲: [`sunmmio_inter_rank_communication_design.md`](./sunmmio_inter_rank_communication_design.md) Section 3.3
   - Pass设计: [`sunmmio_inter_rank_pass_design.md`](./sunmmio_inter_rank_pass_design.md) Section 3.1
   - 并行工作计划: [`sunmmio_inter_rank_parallel_work_plan.md`](./sunmmio_inter_rank_parallel_work_plan.md) Section 5

3. **代码位置**:
   - Frontend: `tilelang/language/dist.py:99-107`
   - 元数据: `src/transform/dist_transform_utils.h:47-81`
   - Pass实现: `src/transform/plan_dist_signals.cc`
   - 测试: `testing/python/sunmmio/inter_rank/test_dist_signal_planning.py`

---

## 🎯 核心概念速查

### 六种Signal Kind

| Kind | TIR名称 | Scope | 更新方式 | 容量 | State dtype | 多sender |
|------|---------|-------|----------|------|-------------|----------|
| SRAM_FLAGREG_INC | `"sram_flagreg_inc"` | SRAM | 硬件+1 | 8 | uint8 | ✅ |
| DRAM_FLAGREG_INC | `"dram_flagreg_inc"` | DRAM | 硬件+1 | 8 | uint8 | ✅ |
| SRAM_FLAGREG_VALUE | `"sram_flagreg_value"` | SRAM | 写generation | 32 | uint32 | ❌ |
| DRAM_FLAGREG_VALUE | `"dram_flagreg_value"` | DRAM | 写generation | 32 | uint32 | ❌ |
| SRAM_MEMORY | `"sram_memory"` | SRAM | 写generation | 不限 | uint32 | ❌ |
| DRAM_MEMORY | `"dram_memory"` | DRAM | 写generation | 不限 | uint32 | ❌ |

### 使用示例

```python
# 1. 自动推断 (推荐)
signal = T.dist.signal()  # kind=None
T.dist.put(src, sram_dst, dst_rank=peer, signal=signal)
# 自动推断为 sram_flagreg_inc

# 2. 显式指定
signal = T.dist.signal(kind=T.dist.SignalKind.DRAM_MEMORY)
T.dist.put(src, dram_dst, dst_rank=peer, signal=signal)
# 强制使用 dram_memory

# 3. 多signal
s0 = T.dist.signal()  # SRAM INC
s1 = T.dist.signal()  # DRAM INC (根据destination推断)
T.dist.put(src, sram_dst, dst_rank=peer, signal=s0)
T.dist.put(src, dram_dst, dst_rank=peer, signal=s1)
```

---

## 🚀 快速上手

### 开发者：我要使用Signal系统

**场景1: Collective Lowering (负责人A)**

```python
# 在LowerDistCollectives中创建signal
signal_handle = Var("collective_signal", DataType::Handle());
Call signal_decl = dist_signal_decl("auto", logical_id++);

# 使用signal
Call put = DistPutOp::make(src, dst, dst_rank, signal_handle);

# PlanDistSignals会自动处理kind推断和index分配
```

**场景2: 访问Signal元数据 (负责人D)**

```cpp
// C++端
const DistSignalKindInfo &kind_info = 
    RequireDistSignalKindInfo(signal_kind_expr, "signal kind");

if (kind_info.update_mode == DistSignalUpdateMode::kIncrement) {
    // INC flagreg: 可以聚合多个sender
    for (auto sender : senders) {
        expected_delta += CountPutsFrom(sender);
    }
} else {
    // VALUE/MEMORY: 检查唯一sender
    ICHECK_EQ(senders.size(), 1U);
}

// 创建state变量时使用正确的dtype
Var generation = Var("gen", kind_info.state_dtype);
```

**场景3: Python测试验证**

```python
def _signal_counts(func):
    return {str(kind): int(count) 
            for kind, count in func.attrs["tl.dist.signal_counts"].items()}

# 验证
assert _signal_counts(device_func) == {
    "sram_flagreg_inc": 2,
    "dram_flagreg_inc": 1,
    "sram_flagreg_value": 0,
    "dram_flagreg_value": 0,
    "sram_memory": 0,
    "dram_memory": 1,
}
```

---

### 开发者：我要扩展Signal系统

**添加新的Signal Kind**

1. 更新枚举和字符串 (`tilelang/language/dist.py`):
```python
class SignalKind(str, Enum):
    # 现有6种...
    NEW_KIND = "new_kind_name"
```

2. 添加元数据 (`src/transform/dist_transform_utils.h`):
```cpp
enum class DistSignalKind {
    // 现有6种...
    kNewKind,
    kCount,  // 保持在最后
};

inline const std::array<DistSignalKindInfo, ...> &
DistSignalKindInfos() {
    static const std::array<DistSignalKindInfo, ...> infos{{
        // 现有6种...
        {DistSignalKind::kNewKind, "new_kind_name",
         DistSignalScope::kSram, DistSignalUpdateMode::kValue,
         16, DataType::UInt(16), false},
    }};
    return infos;
}
```

3. 更新测试 (`testing/python/sunmmio/inter_rank/test_dist_signal_planning.py`):
```python
def test_new_signal_kind():
    signal = T.dist.signal(kind=T.dist.SignalKind.NEW_KIND)
    # 测试逻辑...
```

---

## 🔍 常见问题排查

### Q1: 编译报错 "signal capacity exceeded"

**原因**: 超过了flagreg的硬件容量限制
- SRAM/DRAM INC flagreg: 每个endpoint 8个
- SRAM/DRAM VALUE flagreg: 每个endpoint 32个

**解决方案**:
```python
# ❌ 错误: 9个SRAM INC signal
signals = [T.dist.signal(kind=T.dist.SignalKind.SRAM_FLAGREG_INC) 
           for _ in range(9)]

# ✅ 方案1: 使用Memory signal (不限数量)
signals = [T.dist.signal(kind=T.dist.SignalKind.SRAM_MEMORY) 
           for _ in range(100)]

# ✅ 方案2: 复用INC signal (如果语义允许)
signal = T.dist.signal()  # 自动推断为INC
for i in range(9):
    T.dist.put(src[i], dst[i], dst_rank=peer, signal=signal)
    # INC可以聚合多次put
```

---

### Q2: 编译报错 "inconsistent destination scopes"

**原因**: 同一signal混用了SRAM和DRAM destination

**解决方案**:
```python
# ❌ 错误: 混用scope
signal = T.dist.signal()
T.dist.put(src, sram_dst, dst_rank=peer, signal=signal)  # SRAM
T.dist.put(src, dram_dst, dst_rank=peer, signal=signal)  # DRAM冲突!

# ✅ 方案: 使用两个独立signal
sram_signal = T.dist.signal()  # 推断为sram_flagreg_inc
dram_signal = T.dist.signal()  # 推断为dram_flagreg_inc
T.dist.put(src, sram_dst, dst_rank=peer, signal=sram_signal)
T.dist.put(src, dram_dst, dst_rank=peer, signal=dram_signal)
```

---

### Q3: 我应该用INC还是VALUE flagreg?

**决策树**:

```
是否有多个sender写入同一receiver signal?
├─ 是 → 必须用INC flagreg (硬件原子自增)
│         T.dist.signal()  // 自动推断为INC
│
└─ 否 → 是否需要明确的generation值?
          ├─ 是 → VALUE flagreg或Memory
          │       T.dist.signal(kind=T.dist.SignalKind.DRAM_FLAGREG_VALUE)
          │
          └─ 否 → INC flagreg (最简单)
                  T.dist.signal()  // 推荐
```

**使用建议**:
- 🌟 **默认使用INC** (自动推断): 开销最小，支持多sender
- 🔧 **VALUE flagreg**: 需要明确generation且sender数量≤32
- 📦 **Memory signal**: 超过32个或需要复杂同步逻辑

---

### Q4: Cross-row routing时Memory signal失败

**原因**: 当前Memory signal不支持cross-row转发

**临时方案**:
```python
# ❌ 不支持: Cross-row + Memory signal
signal = T.dist.signal(kind=T.dist.SignalKind.DRAM_MEMORY)
T.dist.put(src, dst, dst_rank=peer, dst_row=other_row, signal=signal)
# Error: Cross-row routing for memory signal not yet supported

# ✅ 临时方案: 使用VALUE flagreg
signal = T.dist.signal(kind=T.dist.SignalKind.DRAM_FLAGREG_VALUE)
T.dist.put(src, dst, dst_rank=peer, dst_row=other_row, signal=signal)
```

**长期解决**: 待底层接口确认后实现 (Section 7.4)

---

## 🛠️ 开发工具

### 调试Pass输出

```python
from testing.python.sunmmio.inter_rank.lowering import lower_to_device_tir

func = kernel_factory.get_tir(M, N, world_size=4)
result = lower_to_device_tir(
    func,
    capture_before_passes="tl.PlanDistSignals",
    capture_passes=("tl.PlanDistSignals", "tl.LowerDistCommunication"),
)

# 查看PlanDistSignals之前的TIR
before = result.pass_snapshot("tl.PlanDistSignals", when="before").mod
print(before.script())
# 应看到: T.dist_signal_decl("auto", ...)

# 查看PlanDistSignals之后的TIR
after = result.pass_snapshot("tl.PlanDistSignals").mod
print(after.script())
# 应看到: T.dist_signal("sram_flagreg_inc", 0)
```

### 验证Signal Counts

```bash
# 运行signal planning测试
pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py -v -k "infers_kinds"

# 检查特定kernel的signal分配
pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py::test_plan_dist_signals_resolves_all_six_explicit_kinds -v -s
```

### 搜索相关代码

```bash
# 查找所有signal kind使用
grep -r "DistSignalKind\|SignalKind" src/ tilelang/ --include="*.cc" --include="*.h" --include="*.py"

# 查找signal_counts属性使用
grep -r "tl.dist.signal_counts" src/ tilelang/ testing/

# 查找PlanDistSignals pass
grep -r "PlanDistSignals" src/transform/ tilelang/
```

---

## 📚 深入学习路径

### Level 1: 用户 (使用Signal API)
1. 阅读: Section 3.3 "Signal与完成语义" (communication_design.md)
2. 运行: `test_dist_signal_planning.py`中的正向测试
3. 实践: 写一个简单的P2P kernel使用signal

### Level 2: 开发者 (理解实现)
1. 阅读: `person_b_signal_planning_implementation_status.md` Section 2
2. 阅读: `src/transform/dist_transform_utils.h` 元数据定义
3. 调试: 设置断点在`PlanDistSignals::VisitStmt_`

### Level 3: 维护者 (扩展系统)
1. 阅读: `person_b_signal_planning_implementation_status.md` 完整文档
2. 阅读: `plan_dist_signals.cc` 完整实现
3. 阅读: Section 7 "已知问题与后续工作"
4. 实践: 添加一个新的signal kind或优化元数据系统

---

## 🤝 协作接口

### 给负责人A (Collective Lowering)

**你需要知道的**:
- ✅ 创建signal时使用`kind=None` (TIR: `"auto"`)
- ✅ `PlanDistSignals`会自动推断和分配index
- ✅ 不需要关心scope校验和容量限制

**示例**:
```cpp
// LowerDistCollectives.cc
Var signal = Var("coll_signal", DataType::Handle());
stmt = SeqStmt({
    Evaluate(dist_signal_decl("auto", logical_id++)),
    DistPutOp::make(src, dst, dst_rank, signal),
    // PlanDistSignals会处理剩余工作
});
```

---

### 给负责人D (Expected计算)

**你需要知道的**:
- ✅ 使用`RequireDistSignalKindInfo()`获取元数据
- ✅ `kind_info.allow_multi_sender`决定是否聚合
- ✅ `kind_info.state_dtype`决定state变量类型

**示例**:
```cpp
// LowerDistCommunication.cc
const DistSignalKindInfo &info = 
    RequireDistSignalKindInfo(signal_kind, "signal kind");

if (info.allow_multi_sender) {
    // INC: 汇总所有sender
    PrimExpr total_delta = Sum(sender_deltas);
} else {
    // VALUE/MEMORY: 检查唯一sender
    ICHECK_EQ(senders.size(), 1U);
}

Var expected = Var("expected_" + info.name, info.state_dtype);
```

---

### 给负责人C (Sender Wait)

**你需要知道的**:
- ✅ Signal planning不处理sender wait
- ✅ `dist_wait_send()`由你的pass负责
- ✅ Signal state只关心receiver端

**注意事项**:
- Signal只表示receiver completion
- Sender completion是独立的本地DMA状态
- 不能用receiver signal wait替代sender wait

---

## 📊 统计数据

### 代码规模
- Frontend: 392 lines (tilelang/language/dist.py)
- 元数据: 335 lines (dist_transform_utils.h)
- Pass实现: 433 lines (plan_dist_signals.cc)
- 测试: 299 lines (test_dist_signal_planning.py)
- **总计**: ~1500 lines

### 测试覆盖
- 测试用例: 13个
- 覆盖率: 92.9%
- 负向测试: 3个 (scope冲突, 容量超限, 混用)
- Pass幂等性: ✅

---

## ✉️ 联系与反馈

**维护人**: 负责人B - 赵世杰 (根据并行工作计划)

**问题反馈**:
1. 新增测试: `testing/python/sunmmio/inter_rank/test_dist_signal_planning.py`
2. 代码审查: 在`plan_dist_signals.cc`提交PR前与负责人D/A同步
3. 设计讨论: 参考`sunmmio_inter_rank_parallel_work_plan.md` Section 8

**相关文档更新**:
- 本指南: 当Signal系统扩展时更新
- 实现状态: 当完成后续优化项时更新
- Pass设计: 当职责边界调整时更新

---

**最后更新**: 2026-09-06  
**版本**: v1.0  
**状态**: ✅ 负责人B核心工作已完成
