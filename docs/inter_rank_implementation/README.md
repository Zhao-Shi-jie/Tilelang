# SunMMIO Inter-Rank Communication 文档集

## 📦 本次文档生成说明

**生成时间**: 2026-09-06  
**任务**: 总结负责人B (赵世杰) Signal资源规划工作的实现状态  
**生成方式**: 基于代码分析、测试覆盖检查、设计文档对比  

---

## 📚 文档清单

### 🆕 本次新增文档 (4个)

| 文档 | 大小 | 用途 | 读者 |
|------|------|------|------|
| **INDEX.md** | 11K | 📑 总索引 - 导航到所有文档 | 所有人 |
| **person_b_signal_planning_implementation_status.md** | 46K | 📊 详细实现报告 - 代码位置、测试覆盖、设计决策 | 开发者、审查者 |
| **README_signal_planning_guide.md** | 12K | 🚀 快速使用指南 - 使用方法、常见问题、协作接口 | 使用者、扩展者 |
| **SUMMARY.md** | 13K | ✅ 工作总结 - 成果、指标、经验 | 项目管理、团队 |
| **QUICK_REFERENCE.md** | 4K | 🔖 1页速查卡 - 打印贴在显示器旁 | 日常开发 |

### 📖 原有设计文档 (4个)

| 文档 | 大小 | 状态 | 说明 |
|------|------|------|------|
| sunmmio_inter_rank_communication_design.md | 44K | ✅ v0.19 | 整体通信设计 |
| sunmmio_inter_rank_pass_design.md | 25K | ✅ v0.4 | Pass设计与现状 |
| sunmmio_inter_rank_parallel_work_plan.md | 16K | ✅ | 并行任务拆分 |
| sunmmio_inter_rank_low_level_overview.md | 9K | ✅ | 底层硬件机制 |

---

## 🎯 快速开始

### 我想了解项目全貌
👉 **阅读**: [INDEX.md](./INDEX.md) → "文档导航"

### 我要查看负责人B的工作完成情况
👉 **阅读**: [SUMMARY.md](./SUMMARY.md) (高层总结) 或 [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) (详细报告)

### 我要使用Signal系统
👉 **阅读**: [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) (1页速查) 或 [README_signal_planning_guide.md](./README_signal_planning_guide.md) (完整指南)

### 我要理解设计原理
👉 **阅读**: [sunmmio_inter_rank_communication_design.md](./sunmmio_inter_rank_communication_design.md) Section 3.3

### 我遇到了问题
👉 **阅读**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "常见问题排查"

---

## 📋 文档结构

```
tmp_docs_dist/
├── README.md (本文件)                     # 文档集说明
├── INDEX.md                               # 总索引
├── QUICK_REFERENCE.md                     # 1页速查卡
├── SUMMARY.md                             # B的工作总结
│
├── 实现文档 (新增) ─────────────────────
│   ├── person_b_signal_planning_implementation_status.md   # 详细实现报告
│   └── README_signal_planning_guide.md                    # 使用指南
│
└── 设计文档 (原有) ─────────────────────
    ├── sunmmio_inter_rank_communication_design.md         # 通信设计
    ├── sunmmio_inter_rank_pass_design.md                  # Pass设计
    ├── sunmmio_inter_rank_parallel_work_plan.md           # 任务分工
    └── sunmmio_inter_rank_low_level_overview.md           # 底层机制
```

---

## 🔍 关键信息速查

### 负责人B (赵世杰) 的工作状态

✅ **已完成** (2026-09-02, PR #370)

**核心成果**:
- ✅ 六种Signal Kind系统
- ✅ 集中元数据管理
- ✅ PlanDistSignals Pass
- ✅ 自动推断逻辑
- ✅ 完整测试覆盖 (92.9%)

**详见**: [SUMMARY.md](./SUMMARY.md) 或 [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md)

---

### 六种Signal Kind速查

| Kind | TIR名称 | 容量 | 多sender |
|------|---------|------|----------|
| SRAM_FLAGREG_INC | `"sram_flagreg_inc"` | 8 | ✅ |
| DRAM_FLAGREG_INC | `"dram_flagreg_inc"` | 8 | ✅ |
| SRAM_FLAGREG_VALUE | `"sram_flagreg_value"` | 32 | ❌ |
| DRAM_FLAGREG_VALUE | `"dram_flagreg_value"` | 32 | ❌ |
| SRAM_MEMORY | `"sram_memory"` | ∞ | ❌ |
| DRAM_MEMORY | `"dram_memory"` | ∞ | ❌ |

**详见**: [QUICK_REFERENCE.md](./QUICK_REFERENCE.md)

---

### 关键文件位置

| 类别 | 文件 |
|------|------|
| Frontend | `tilelang/language/dist.py:99-107` |
| 元数据 | `src/transform/dist_transform_utils.h:47-81` |
| Pass实现 | `src/transform/plan_dist_signals.cc` |
| 测试 | `testing/python/sunmmio/inter_rank/test_dist_signal_planning.py` |

**详见**: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) Section 2

---

## 🚀 下一步行动

### 对于项目管理者

1. ✅ **确认B的工作**: 阅读 [SUMMARY.md](./SUMMARY.md)
2. 📋 **规划后续任务**: 参考 [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 3-7
3. 🔄 **启动并行开发**: 负责人A/C/D可以基于B的成果并行推进

### 对于负责人A (Collective)

1. 📖 **阅读协作接口**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "给负责人A"
2. 💻 **开始开发**: 使用`T.dist.signal()`创建signal，PlanDistSignals会自动处理
3. 📝 **参考示例**: [test_dist_signal_planning.py](testing/python/sunmmio/inter_rank/test_dist_signal_planning.py)

### 对于负责人D (Expected)

1. 📖 **阅读元数据API**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "给负责人D"
2. 💻 **使用元数据**: 
   ```cpp
   const DistSignalKindInfo &info = 
       RequireDistSignalKindInfo(signal_kind_expr, "signal kind");
   ```
3. 📝 **参考实现**: [dist_transform_utils.h](src/transform/dist_transform_utils.h:47-105)

### 对于负责人C (Sender Wait)

1. 📖 **理解职责边界**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "给负责人C"
2. 💻 **独立开发**: Signal planning不处理sender wait，C可以独立推进
3. 📝 **参考设计**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 6

---

## 📊 文档统计

### 总体规模
- **文档总数**: 9个 (5新增 + 4原有)
- **总大小**: ~180KB
- **代码引用**: 50+ 代码片段
- **测试用例**: 13个

### 覆盖范围
- ✅ 设计原理
- ✅ 实现细节
- ✅ 使用方法
- ✅ 测试验证
- ✅ 问题排查
- ✅ 协作接口
- ✅ 项目管理

---

## 🛠️ 文档维护

### 何时更新本README

1. **新增文档时**: 在"文档清单"添加条目
2. **工作进展变化**: 更新"负责人B的工作状态"
3. **关键信息变化**: 更新"关键信息速查"

### 何时更新其他文档

参见: [INDEX.md](./INDEX.md) → "文档维护指南"

---

## ✉️ 反馈与贡献

### 发现问题
- 文档错误或不清楚的地方 → 在对应文档标注问题位置
- 代码与文档不一致 → 优先以代码为准，更新文档

### 贡献改进
- 新增使用示例 → 补充到 [README_signal_planning_guide.md](./README_signal_planning_guide.md)
- 新的常见问题 → 补充到 [README_signal_planning_guide.md](./README_signal_planning_guide.md) "常见问题排查"
- 设计变更 → 更新对应设计文档

---

## 📞 联系方式

**文档生成**: 自动化分析工具  
**负责人B**: 赵世杰  
**项目**: SunMMIO Rank间通信  

---

**最后更新**: 2026-09-06  
**版本**: v1.0  
**状态**: ✅ 完整
