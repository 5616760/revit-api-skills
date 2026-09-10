---
name: revit-2d-view-creation-paths
description: |
  当需要通过 API 创建平面/剖面/详图/索引/立面/图纸视图时调用。
  不适用于：三维视图创建、视图分类判别。
  关键 trigger 信号："创建平面视图 create plan view"、"剖面 section view"、"立面 elevation"、"图纸 sheet view"、"ElevationMarker 两步走"。
  核心认知：每类视图有不同创建入口与先决条件，立面需先建 ElevationMarker 再 CreateElevation。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.6.2节视图类型 (约p126-133)
tags: [2d-view, plan-view, section-view, elevation, sheet, revit-api]
related_skills:
  - slug: revit-view-classification-paths
    relation: composes-with
---

# 平面/剖面/立面/图纸视图创建路径

## R — 原文 (Reading)

> 平面视图基于标高……ViewPlan.Create() 创建楼板和天花板平面……ViewSection.CreateSection() 创建剖面……要创建立面视图，首先要创建一个立面符号，再用此符号生成立面视图……ViewSheet.Create() 创建图纸视图。
>
> — 宦国胜, 第2章 2.6.2节视图类型 (约p126-133)

---

## I — 方法论骨架 (Interpretation)

每类二维视图有不同创建入口与不同先决条件，构成一张路径表：

- **平面视图**：`ViewPlan.Create(doc, viewFamilyTypeId, levelId)` 基于标高；面积平面用 `ViewPlan.CreateAreaPlan(doc, areaSchemeId, levelId)`。先决条件：ViewFamilyTypeId + LevelId。
- **剖面视图**：`ViewSection.CreateSection(doc, viewFamilyTypeId, sectionBox)`。先决条件：ViewFamilyTypeId + BoundingBoxXYZ sectionBox。
- **详图视图**：`ViewSection.CreateDetail(doc, viewFamilyTypeId, sectionBox)`。
- **索引视图**：`ViewSection.CreateCallout(doc, parentViewId, viewFamilyTypeId, ...)`。
- **立面视图（特殊两步）**：先 `doc.Create.NewElevationMarker(...)` 创建立面符号，再 `marker.CreateElevation(doc, viewFamilyTypeId, ...)` 生成立面视图——立面区别于所有其他视图类型的结构性差异。
- **图纸视图**：`ViewSheet.Create(doc, titleBlockId)` → `Viewport.Create(doc, viewSheetId, viewId, point)` → `viewSheet.Print()`。同一视图不能上多张图纸（`CanAddViewToSheet` 可前置校验）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 立面视图的两步创建

- **问题**: 为什么不能像创建剖面那样直接创建立面视图？
- **方法论的使用**: 立面视图的几何由立面符号（ElevationMarker）决定，必须先 CreateElevationMarker 再 marker.CreateElevation——这是立面区别于所有其他视图类型的结构性差异
- **结论**: 立面必须两步走，不能直接创建
- **结果**: 成功创建立面视图

### 案例 2: 图纸批量排版与打印

- **问题**: 图纸批量排版后如何打印？
- **方法论的使用**: 打印走 `viewSheet.Print()`，但视图必须先经 `Viewport.Create` 放上图纸，且同一视图不能上多张图纸（`CanAddViewToSheet` 前置校验）
- **结论**: 图纸-视口-打印三段链路
- **结果**: 批量排版后成功打印

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要批量创建平面/剖面/立面视图
2. 立面视图创建失败，需要理解两步走的特殊性
3. 需要批量排版图纸并打印
4. 同一视图上多张图纸时遇到 CanAddViewToSheet 失败

### 语言信号 (用户的话里出现这些就应激活)

- "创建平面视图 / create plan view / ViewPlan Create"
- "剖面视图 / section view / ViewSection CreateSection"
- "立面视图 / elevation view / ElevationMarker"
- "图纸视图 / sheet view / ViewSheet Create"
- "两步走 / two-step creation / 立面符号"

### 与相邻 skill 的区分

- 与 `revit-view-classification-paths`：本 skill 依赖其分类路径获取 ViewFamilyTypeId 作为创建前置，组合成"分类→创建"流程；本 skill 负责平面/剖面/立面/图纸的具体入口，后者不涉及创建。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定目标视图类型并获取对应先决条件**
   - 平面：ViewFamilyTypeId + LevelId
   - 剖面/详图：ViewFamilyTypeId + BoundingBoxXYZ
   - 立面：先 ElevationMarker
   - 图纸：TitleBlockId
   - 完成标准: 明确视图类型和先决条件

2. **按对应入口创建视图**
   - 平面：`ViewPlan.Create(doc, viewFamilyTypeId, levelId)`
   - 剖面：`ViewSection.CreateSection(doc, viewFamilyTypeId, sectionBox)`
   - 立面：`ElevationMarker marker = doc.Create.NewElevationMarker(...); marker.CreateElevation(doc, viewFamilyTypeId, ...);`
   - 图纸：`ViewSheet sheet = ViewSheet.Create(doc, titleBlockId);`
   - 完成标准: 视图创建成功
   - 判停条件: 若立面直接调 CreateElevation 会失败，必须先建 ElevationMarker

3. **图纸场景下放置视口并打印**
   - `Viewport.Create(doc, sheet.Id, viewId, point);`
   - 前置校验 `sheet.CanAddViewToSheet(doc, viewId)` 防止重复
   - `sheet.Print();`
   - 完成标准: 视图放上图纸并打印

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 三维视图创建——走 CreateIsometric/CreatePerspective
- 视图分类判别——走 revit-view-classification-paths

### 作者在书中警告的失败模式

- 立面视图不能直接创建——必须先建 ElevationMarker 再 CreateElevation
- 同一视图不能上多张图纸——CanAddViewToSheet 返回 false

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增视图创建 API 或简化立面两步走流程

### 容易混淆的邻近方法论

- 立面两步走（ElevationMarker → CreateElevation）vs 剖面一步走（CreateSection）——立面需符号前置
- 图纸（ViewSheet）vs 视口（Viewport）——前者是图纸视图，后者是视图在图纸上的放置

---

## 相关 skills

- **revit-view-classification-paths**（composes-with）：创建二维视图前先经分类路径取得 ViewFamilyTypeId，二者组合为"分类→创建"链路。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
