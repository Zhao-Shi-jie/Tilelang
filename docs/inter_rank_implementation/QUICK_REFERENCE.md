# Signal资源规划 - 快速参考卡

> 1页纸速查表 - 打印后贴在显示器旁

---

## 📋 六种Signal Kind速查

| Frontend | TIR String | Scope | 更新 | 容量 | dtype | 多sender |
|----------|-----------|-------|------|------|-------|----------|
| `SRAM_FLAGREG_INC` | `"sram_flagreg_inc"` | SRAM | 硬件+1 | 8 | uint8 | ✅ |
| `DRAM_FLAGREG_INC` | `"dram_flagreg_inc"` | DRAM | 硬件+1 | 8 | uint8 | ✅ |
| `SRAM_FLAGREG_VALUE` | `"sram_flagreg_value"` | SRAM | 写gen | 32 | uint32 | ❌ |
| `DRAM_FLAGREG_VALUE` | `"dram_flagreg_value"` | DRAM | 写gen | 32 | uint32 | ❌ |
| `SRAM_MEMORY` | `"sram_memory"` | SRAM | 写gen | ∞ | uint32 | ❌ |
| `DRAM_MEMORY` | `"dram_memory"` | DRAM | 写gen | ∞ | uint32 | ❌ |

---

## 🚀 常用代码片段

### 1. 自动推断 (推荐)
```python
signal = T.dist.signal()  # kind=None
T.dist.put(src, sram_dst, dst_rank=peer, signal=signal)
# → 自动推断为 sram_flagreg_inc
```

### 2. 显式指定
```python
signal = T.dist.signal(kind=T.dist.SignalKind.DRAM_MEMORY)
T.dist.put(src, dram_dst, dst_rank=peer, signal=signal)
```

### 3. 多signal独立
```python
s0 = T.dist.signal()  # SRAM INC
s1 = T.dist.signal()  # DRAM INC
T.dist.put(src, sram_dst, dst_rank=peer, signal=s0)
T.dist.put(src, dram_dst, dst_rank=peer, signal=s1)
```

### 4. 同一signal多次put (INC)
```python
signal = T.dist.signal()  # INC允许聚合
for i in range(N):
    T.dist.put(src[i], dst[i], dst_rank=peer, signal=signal)
# Receiver: expected = N
```

---

## ⚠️ 常见错误

### ❌ 错误1: 混用scope
```python
signal = T.dist.signal()
T.dist.put(src, sram_dst, signal=signal)  # SRAM
T.dist.put(src, dram_dst, signal=signal)  # DRAM → 冲突!
```
**修复**: 使用两个独立signal

### ❌ 错误2: 超过容量
```python
# 9个SRAM INC signal (最多8个)
for i in range(9):
    signals.append(T.dist.signal(kind=T.dist.SignalKind.SRAM_FLAGREG_INC))
```
**修复**: 使用Memory signal或复用INC

### ❌ 错误3: Scope不匹配
```python
signal = T.dist.signal(kind=T.dist.SignalKind.SRAM_FLAGREG_INC)
T.dist.put(src, dram_dst, signal=signal)  # SRAM signal + DRAM dst → 冲突!
```
**修复**: 使用`DRAM_FLAGREG_INC`或自动推断

---

## 🔍 调试命令

### 查看Pass输出
```python
result = lower_to_device_tir(func, capture_passes="tl.PlanDistSignals")
print(result.pass_snapshot("tl.PlanDistSignals").mod.script())
```

### 检查signal_counts
```python
def _signal_counts(func):
    return {str(k): int(v) for k, v in func.attrs["tl.dist.signal_counts"].items()}
```

### 运行测试
```bash
pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py -v
```

---

## 📂 关键文件位置

| 类别 | 文件 |
|------|------|
| Frontend枚举 | `tilelang/language/dist.py:99-107` |
| 元数据定义 | `src/transform/dist_transform_utils.h:47-81` |
| Pass实现 | `src/transform/plan_dist_signals.cc` |
| 测试 | `testing/python/sunmmio/inter_rank/test_dist_signal_planning.py` |

---

## 🎯 选择建议

```
需要多sender写入同一signal?
├─ 是 → INC flagreg (T.dist.signal())
└─ 否 → 需要明确generation?
          ├─ 是 → VALUE flagreg或Memory
          └─ 否 → INC flagreg (默认)
```

**经验法则**:
- 🌟 **90%场景**: 使用`T.dist.signal()` (自动推断INC)
- 🔧 **特殊需求**: 显式指定VALUE或MEMORY
- 📦 **超大规模**: 使用Memory signal (不限数量)

---

## 🆘 求助资源

| 问题类型 | 查阅文档 |
|----------|----------|
| 使用方法 | [README_signal_planning_guide.md](./README_signal_planning_guide.md) |
| 实现细节 | [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) |
| 设计原理 | [sunmmio_inter_rank_communication_design.md](./sunmmio_inter_rank_communication_design.md) |
| Pass设计 | [sunmmio_inter_rank_pass_design.md](./sunmmio_inter_rank_pass_design.md) |

---

**版本**: v1.0 | **更新**: 2026-09-06 | **负责人**: 赵世杰
