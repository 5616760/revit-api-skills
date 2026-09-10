---
name: revit-analysis-results-refresh
description: |
  做分析可视化插件（能量分析、SpatialFieldManager、结构/能耗结果着色）时调用：Revit 分析框架
  不会自动更新结果，模型任何更改都可能使结果无效，必须主动刷新——用 IUpdater（DMU，事务关闭前
  介入，修改并入原事务）或订阅 DocumentChanged（事务关闭后通知，重算再提交新事务），二选一。
  Trigger："SpatialFieldManager"、"分析结果不刷新"、"模型变了结果还在"、"energy analysis
  refresh"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章5.11.2节分析可视化（约p389）
tags: [analysis-visualization, dmu, spatialfieldmanager, revit-api]
related_skills:
  - slug: revit-iupdater-execute-transaction-rules
    relation: depends-on
  - slug: revit-readonly-event-checks
    relation: depends-on
---

# 分析成果更新需用动态模型更新或事件触发

## R — 原文 (Reading)

> "Revit 分析框架不会自动更新结果，对 Revit 模型的任何更改都可能使结果无效。当 Revit 模型已发生更改、先前的计算结果可能无效，需要重新计算时，为了使结果能够更新，API 开发人员应使用动态模型更新触发器或订阅 DocumentChanged 事件等待更新通知。"
>
> — 宦国胜，第5章5.11.2节分析可视化（约p389）

---

## I — 方法论骨架 (Interpretation)

Revit 的分析结果（SpatialFieldManager 承载的空间场数据等）是"快照"而非"实时派生"——模型变了它不会自己重算。

- 隐含前提：任何模型更改（几何、参数、删除图元）都可能导致已有分析结果失效，但 Revit 既不标记失效也不自动刷新。
- 两条刷新路径：
  1. **IUpdater（DMU）**：注册触发器（如 `Element.GetChangeTypeGeometry()`），模型变更事务收尾前 Execute 中重算并写回结果——结果更新与模型修改并入同一撤销单元，一致性强。
  2. **DocumentChanged 事件**：事务关闭后收到通知，回调中重算——需要开新事务提交结果，模型与结果在撤销历史上是两步。
- 选型规则：需要"结果与模型同步撤销/重做"（同一事务）→ DMU；只需要"事后刷新"（可接受两步撤销、或计算昂贵想批量做）→ 事件。
- 共同点：都不能指望框架自动算，"何时失效 + 如何重算"完全是开发者的责任。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 结构分析结果可视化插件

- **问题**: 模型修改后，墙/梁上的分析结果着色仍显示旧值。
- **方法论的使用**: 按 5.11.2 的结论——实现 IUpdater 注册几何变更触发器，Execute 中重新计算 SpatialFieldManager 的分析结果；或订阅 DocumentChanged 后重算。
- **结论**: 失效检测与刷新必须自己搭，两条路径按事务需求二选一。
- **结果**: 模型修改后结果自动重算，且撤销模型修改时结果（DMU 路径）一并回滚。

### 案例 2: 能量分析结果随模型更新

- **问题**: 用户调整了房间边界，能量分析数据未跟随变化。
- **方法论的使用**: 用 DocumentChanged 监听变更，回调中触发重算（计算昂贵，接受两步撤销）。
- **结论**: 事件路径适合"重、可延迟"的计算。
- **结果**: 结果在事务提交后刷新，模型操作保持轻快。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开发分析可视化功能（着色、云图、热力图）时设计刷新机制。
2. 用户报告"改了模型但分析结果没变/还显示旧数据"。
3. 权衡"结果随模型一起撤销"还是"分步撤销"的事务设计。

### 语言信号 (用户的话里出现这些就应激活)

- "SpatialFieldManager / AnalysisResultManager"
- "分析结果 不刷新 / 自动更新"（analysis results not refreshing / auto-update）
- "能量分析 能耗 可视化"（energy analysis visualization）

### 与相邻 skill 的区分

- 与 `revit-iupdater-execute-transaction-rules` 的关系：本 skill 在“DMU 更新器刷新”路径下复用该 skill 的事务规则，保证结果与模型一致。
- 与 `revit-readonly-event-checks` 的关系：本 skill 在“事件刷新”路径下复用该 skill 的只读检查，避免在只读事件中修改分析结果。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定刷新一致性要求**
   - 问：结果必须与模型修改同一撤销单元吗？
   - 完成标准: 明确写出"同一事务（DMU）/两步（事件）"的选择及理由。
   - 判停条件: 若计算极重或需外部引擎，固定选事件路径（甚至手动刷新按钮），跳过 DMU 评估。

2. **实现所选路径**
   - DMU：注册针对分析相关图元的触发器，Execute 中重算并写 SpatialFieldManager。
   - 事件：订阅 DocumentChanged，回调（注意可写性检查）中重算并开新事务提交。
   - 完成标准: 模型修改后结果在一个交互周期内刷新。

3. **验证撤销/重做一致性**
   - 完成标准: 撤销模型修改后，结果状态符合第 1 步的选型预期（同回滚或独立两步）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 静态一次性分析报告（导出后不再展示）——无需自动刷新机制。
- 外部引擎主导的计算（耗时分钟级）——自动重算会卡死交互，改用显式"重新计算"命令。

### 作者在书中警告的失败模式

- 假设 Revit 自动刷新分析结果 → 模型改了结果还是旧值（本单元本体）。
- 结果数据长期不失效检测 → 用户基于过期结果做决策。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版能量分析框架有变化（部分走向云端/服务化），但 SpatialFieldManager 的"不自动刷新"设计延续。

### 容易混淆的邻近方法论

- 视图刷新（doc.Regenerate / 视图参数）≠ 分析结果重算——前者只是图形重建，后者是数据重计算。

---

## 相关 skills

- **revit-iupdater-execute-transaction-rules**（IUpdater Execute 方法约束与事务规则 · depends-on）— 选 DMU 路径时 Execute 内改分析结果需遵循该 skill 的事务规则。
- **revit-readonly-event-checks**（只读事件与模型修改检查原则 · depends-on）— 事件路径里改分析结果前需通过该 skill 的只读检查。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
