---
name: revit-level-no-auto-view
description: |
  当 API 创建标高后发现没有对应平面视图，或需理解 API 与 UI 隐式行为差异时调用。
  不适用于：UI 中创建标高、直接创建平面视图。
  关键 trigger 信号："创建标高没有平面视图 create level no plan view"、"NewLevel 不生成视图"、"批量创建标高 batch create levels"、"UI 与 API 行为差异 UI vs API"。
  核心：NewLevel 只生成 Level 图元，不生成关联平面视图——需显式调 ViewPlan.Create 补建。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.5.1 标高 (p196-197)
tags: [level, plan-view, api-ui-difference, newlevel, revit-api]
related_skills:
  - slug: revit-2d-view-creation-paths
    relation: depends-on
---

# 创建新标高不会自动创建关联平面视图

## R — 原文 (Reading)

> 在创建新标高后，Revit 不会为此标高创建相关联的平面视图。如果需要，用户可以自己创建。
>
> — 宦国胜, 第3章 3.5.1 标高 (p196-197)

---

## I — 方法论骨架 (Interpretation)

这是 Revit 二次开发中 API 与 UI 行为差异的典型认知陷阱：

**UI 行为**：在 Revit 用户界面中创建标高时，通常会弹出询问框"是否要创建对应的平面视图"，用户选择后会自动生成关联的楼层/天花板平面视图。这暗示标高与平面视图是绑定的。

**API 行为**：`Document.Create.NewLevel(elevation)` 只生成 Level 图元本身，不生成任何关联的平面视图——API 静默跳过了 UI 的询问环节。这并非 bug，而是设计意图：API 只做显式请求的事。

**正确做法**：在 `NewLevel` 后显式调用 `ViewPlan.Create(document, viewFamilyTypeId, levelId)` 补建平面视图。批量创建标高的工具若漏这一步，用户会以为标高创建失败。

这种"API 与 UI 隐式行为差异"在 Revit 中是普遍模式——API 不隐式创建关联图元，所有关联操作必须显式调用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量创建标高工具

- **问题**: 批量创建 100 个标高的工具，用户抱怨没有对应平面视图，为什么？
- **方法论的使用**: 识别 API 与 UI 的隐式行为差异——NewLevel 不自动创建视图（与 UI 行为不同）。工具需在 NewLevel 后显式调 `ViewPlan.Create(document, viewFamilyTypeId, levelId)` 补建平面视图
- **结论**: API 只做显式请求的事，关联视图需分别显式创建
- **结果**: 补建视图后用户不再抱怨

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批量创建标高后发现没有对应平面视图
2. 编写创建标高的插件，需要理解 API 不会自动创建视图
3. 需要理解 API 与 UI 的隐式行为差异
4. 标高创建后需要补建楼层/天花板平面视图

### 语言信号 (用户的话里出现这些就应激活)

- "创建标高没有平面视图 / create level no plan view / NewLevel 不生成视图"
- "API 不自动创建视图 / API doesn't auto-create view"
- "批量创建标高 / batch create levels / 100 个标高"
- "UI 与 API 行为差异 / UI vs API behavior difference"

### 与相邻 skill 的区分

- 与 `revit-2d-view-creation-paths`：本 skill 依赖后者提供的 ViewPlan.Create 等创建入口来补建标高关联视图；本 skill 聚焦"API 不自动建视图"这一隐式行为，后者聚焦二维视图创建入口本身。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **创建标高**
   - `Level level = doc.Create.NewLevel(elevation);`
   - 完成标准: Level 图元创建成功（注意此时无关联平面视图）

2. **显式创建关联平面视图**
   - 获取 ViewFamilyType（楼层平面类型）
   - `ViewPlan.Create(document, viewFamilyTypeId, level.Id);`
   - 完成标准: 平面视图创建成功并与标高关联
   - 判停条件: 若 ViewPlan.Create 返回 null，检查 ViewFamilyType 是否正确、Level 是否有效

3. **验证标高与视图关联**
   - 确认平面视图的 LevelId == 新建标高的 Id
   - 完成标准: 标高与平面视图正确关联

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- UI 中创建标高——会弹询问框自动创建视图
- 直接创建平面视图（不通过标高）——直接调 ViewPlan.Create

### 作者在书中警告的失败模式

- API 不隐式创建关联图元——批量工具漏补建视图会导致用户误以为标高创建失败
- UI 经验暗示标高与平面视图绑定，但 API 中它们是独立图元

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增 API 选项自动创建关联视图

### 容易混淆的邻近方法论

- API 的 NewLevel（不弹询问框）vs UI 的创建标高（弹询问框）——API 静默跳过 UI 交互
- 标高（Level 图元）vs 平面视图（ViewPlan 图元）——两者是独立图元，需分别创建

---

## 相关 skills

- **revit-2d-view-creation-paths**（depends-on）：本 skill 的"补建平面视图"步骤需调用 ViewPlan.Create 等创建入口，依赖该 skill 的二维视图创建方法。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
