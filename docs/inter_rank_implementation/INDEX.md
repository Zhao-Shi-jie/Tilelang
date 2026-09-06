# SunMMIO Inter-Rank Communication 文档索引

> 最后更新: 2026-09-06

## 📚 文档导航

### 🎯 核心设计文档

| 文档 | 描述 | 读者 | 状态 |
|------|------|------|------|
| [sunmmio_inter_rank_communication_design.md](./sunmmio_inter_rank_communication_design.md) | **通信设计大纲** - 整体架构、frontend语法、TIR表示、pass pipeline | 所有开发者 | ✅ v0.19 |
| [sunmmio_inter_rank_pass_design.md](./sunmmio_inter_rank_pass_design.md) | **Pass设计与现状** - 四个dist pass的职责、输入输出、依赖关系 | Pass开发者 | ✅ v0.4 |
| [sunmmio_inter_rank_low_level_overview.md](./sunmmio_inter_rank_low_level_overview.md) | **底层机制概览** - PCIe DMA、flagreg、doorbell、物理地址 | 底层实现、SUVM对接 | ✅ |
| [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) | **并行任务拆分** - 负责人A/B/C/D的工作划分、交付标准 | 项目管理、协作 | ✅ v0.x |

---

### 📋 实现状态文档 (新增)

| 文档 | 描述 | 读者 | 状态 |
|------|------|------|------|
| [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) | **负责人B实现状态报告** - 详细的完成情况、测试覆盖、代码位置、设计细节 | 所有开发者、项目审查 | ✅ v1.0 (2026-09-06) |
| [README_signal_planning_guide.md](./README_signal_planning_guide.md) | **Signal资源规划快速指南** - 使用手册、常见问题、协作接口 | Signal系统使用者、扩展者 | ✅ v1.0 (2026-09-06) |

---

## 🚀 快速开始

### 我是新加入的开发者

**5分钟快速上手**:
1. 阅读: [README_signal_planning_guide.md](./README_signal_planning_guide.md) 的"核心概念速查"
2. 阅读: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 2 "当前共识"
3. 运行: `pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py -v`

**30分钟深入理解**:
1. 阅读: [sunmmio_inter_rank_communication_design.md](./sunmmio_inter_rank_communication_design.md) Section 1-3
2. 阅读: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) Section 2
3. 调试: 设置断点在`PlanDistSignals`并单步执行

---

### 我要查找特定信息

#### Signal相关
- **Signal Kind定义**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "六种Signal Kind"表格
- **使用示例**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "快速上手"
- **实现细节**: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) → Section 2
- **设计原理**: [sunmmio_inter_rank_communication_design.md](./sunmmio_inter_rank_communication_design.md) → Section 3.3
- **底层机制**: [sunmmio_inter_rank_low_level_overview.md](./sunmmio_inter_rank_low_level_overview.md) → Section 4

#### Pass相关
- **Pass职责**: [sunmmio_inter_rank_pass_design.md](./sunmmio_inter_rank_pass_design.md) → Section 3
- **Pass顺序**: [sunmmio_inter_rank_pass_design.md](./sunmmio_inter_rank_pass_design.md) → Section 2
- **Pass输入输出**: [sunmmio_inter_rank_pass_design.md](./sunmmio_inter_rank_pass_design.md) → 各pass子节
- **调试方法**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "开发工具"

#### 任务分工
- **整体分工**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) → Section 3
- **负责人A (Collective)**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) → Section 4
- **负责人B (Signal)**: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) → 完整文档
- **负责人C (Sender Wait)**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) → Section 6
- **负责人D (Expected)**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) → Section 7

#### 问题排查
- **编译错误**: [README_signal_planning_guide.md](./README_signal_planning_guide.md) → "常见问题排查"
- **测试失败**: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) → Section 3 "交付验收"
- **已知限制**: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md) → Section 7 "已知问题"

---

## 📊 项目进度概览

### 负责人A - 杨晓朝 (Collective Frontend与Lowering)
- **状态**: 🟡 设计确认，待实现
- **核心产出**: `LowerDistCollectives` pass, `all_gather`/`all_to_all`/`all_reduce`
- **依赖**: 复用B/D的基础合同
- **文档**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 4

### 负责人B - 赵世杰 (Signal资源规划) ✅
- **状态**: ✅ **已完成**
- **核心产出**: 六种Signal Kind、`PlanDistSignals` pass、元数据系统
- **完成时间**: 2026-09-02 (PR #370)
- **文档**: 
  - 详细状态: [person_b_signal_planning_implementation_status.md](./person_b_signal_planning_implementation_status.md)
  - 使用指南: [README_signal_planning_guide.md](./README_signal_planning_guide.md)

### 负责人C - 肖尧 (Sender Completion Wait)
- **状态**: 🟡 设计确认，待实现
- **核心产出**: `InsertDistSenderWait` pass (独立文件)
- **依赖**: B的signal planning完成后可并行开发
- **文档**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 6

### 负责人D - 关舟 (Receiver Wait与Expected)
- **状态**: 🟡 设计确认，待实现
- **核心产出**: `InjectDistSync`中的generation/expected计算
- **依赖**: 消费B的resolved kind元数据
- **文档**: [sunmmio_inter_rank_parallel_work_plan.md](./sunmmio_inter_rank_parallel_work_plan.md) Section 7

---

## 🏗️ 代码结构

```
src/
├── op/
│   ├── dist_comm.h             # Dist op定义
│   └── dist_comm.cc            # Dist op实现
└── transform/
    ├── dist_transform_utils.h  # ✅ 六种Signal Kind元数据 (负责人B)
    ├── plan_dist_signals.cc    # ✅ Signal资源规划 (负责人B)
    ├── lower_dist_routing.cc   # ✅ 路由降级 (基础设施)
    ├── lower_dist_communication.cc  # 🟡 Expected计算 (负责人D待完善)
    └── inject_dist_sync.cc     # 🟡 Generation更新 (负责人D待完善)

tilelang/
├── language/
│   └── dist.py                 # ✅ Frontend语法、六种SignalKind (负责人B)
└── transform/
    └── __init__.py             # Pass注册

testing/python/sunmmio/inter_rank/
├── test_dist_signal_planning.py    # ✅ Signal planning测试 (负责人B)
├── test_dist_row_routing.py        # ✅ 路由测试
├── test_dist_stage3_completion.py  # ✅ 完整pipeline测试
└── test_dist_foundation.py         # ✅ 基础功能测试
```

---

## 📈 版本历史

### v0.19 - 2026-09-02 (PR #370)
- ✅ **负责人B工作完成**: 六种Signal Kind、PlanDistSignals pass、元数据系统
- ✅ 基础P2P DSL和TIR lowering pipeline
- ✅ 四个dist pass的基础实现
- ✅ 完整的测试覆盖
- **PR**: #370 (commit 09373e39)

### v0.18 - 2026-09-01 (设计确认)
- Signal Kind扩展为六个完整枚举
- State dtype按kind区分 (uint8/uint32)
- Multi-sender规则按update mode区分

### 阶段三 (v0.17) - 2026-08-28
- Cross-row routing
- 多signal独立状态
- 静态P2P收口

### 阶段二 (v0.13-0.16) - 2026-08-26
- 最小闭环: put + signal + wait_signal

### 阶段一 - 2026-08-20
- 基础语法搭建: world_size, RankId, rank_placement

---

## 🎯 后续计划

### 优先级P0 (不阻塞，持续优化)
- [ ] 将payload validation从B移到routing (Section 7.1)
- [ ] 清理旧signal_counts格式兼容代码
- [ ] 补充plan_dist_signals.cc内部注释

### 优先级P1 (依赖B完成，可并行)
- [ ] **负责人D**: 完善InjectDistSync的generation/expected
- [ ] **负责人C**: 实现InsertDistSenderWait自动插入
- [ ] **负责人A**: 实现LowerDistCollectives (all_gather/all_to_all)

### 优先级P2 (依赖外部信息)
- [ ] 确认VALUE flagreg实际位宽
- [ ] 实现Memory signal的cross-row支持
- [ ] SUVM codegen对接

---

## 🔗 外部资源

### 代码仓库
- **主仓库**: `/workspace/Tilelang`
- **关键分支**: `dev/dist` (开发), `main` (稳定)
- **关键PR**: #370 (基础实现)

### 底层参考
- **compiler-samples**: `samples/31_compute_exchange/`, `samples/33_ring_allgather_dram/`
- **SUVM helpers**: `common/su_pcie_dma.h`, `common/su_flag_reg.h`

### 测试命令
```bash
# 运行所有inter-rank测试
pytest testing/python/sunmmio/inter_rank/ -v

# 只运行signal planning测试
pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py -v

# 调试模式
pytest testing/python/sunmmio/inter_rank/test_dist_signal_planning.py::test_plan_dist_signals_infers_kinds_and_preserves_independent_state -v -s
```

---

## 📝 文档维护指南

### 何时更新本INDEX

1. **新增文档时**:
   - 在"📚 文档导航"添加条目
   - 在"我要查找特定信息"添加索引链接

2. **项目进度变化时**:
   - 更新"📊 项目进度概览"的状态标记
   - 更新"🏗️ 代码结构"的完成标记

3. **重要版本发布时**:
   - 在"📈 版本历史"添加记录
   - 更新"🎯 后续计划"

### 何时更新其他文档

| 文档 | 更新触发条件 |
|------|-------------|
| communication_design.md | 设计变更、新功能支持决策 |
| pass_design.md | Pass职责调整、新增pass |
| parallel_work_plan.md | 任务重新分配、交付标准变更 |
| person_b_*.md | B的工作有重大变更或后续优化完成 |
| README_signal_planning_guide.md | Signal使用方式变更、新增常见问题 |

---

## ✉️ 联系方式

**项目负责人**:
- 负责人A (Collective): 杨晓朝
- 负责人B (Signal): 赵世杰 ✅
- 负责人C (Sender Wait): 肖尧
- 负责人D (Expected): 关舟

**问题反馈**:
1. 设计问题 → 在对应设计文档的issue tracker
2. 实现问题 → 在具体PR或commit留言
3. 使用问题 → 参考README_signal_planning_guide.md的"常见问题排查"

---

**索引维护人**: 文档自动生成系统  
**最后更新**: 2026-09-06  
**版本**: v1.0  

---

## 🔖 快捷链接

- [🎯 负责人B实现状态](./person_b_signal_planning_implementation_status.md) ← **最新完成**
- [📘 Signal使用指南](./README_signal_planning_guide.md) ← **推荐阅读**
- [🏗️ 通信设计大纲](./sunmmio_inter_rank_communication_design.md)
- [⚙️ Pass设计](./sunmmio_inter_rank_pass_design.md)
- [👥 并行工作计划](./sunmmio_inter_rank_parallel_work_plan.md)
- [🔧 底层机制](./sunmmio_inter_rank_low_level_overview.md)
