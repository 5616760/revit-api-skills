---
name: revit-referencepoint-curvebypoints
description: |
  用户要在概念设计/体量环境中创建参照点、用点创建曲线/形状、在曲线指定位置（如 25%）精确放点、或希望移动驱动点后形状跟随更新时调用。不适用于：项目文档创建普通图元（参照点仅概念设计环境）、固定几何无需参数化（用模型线）。关键 trigger："概念设计环境"、"ReferencePoint / PointOnEdge"、"在曲线上 25% 处放一个点"、"移动点形状跟着变"、"CurveByPoints"。核心：参照点是参数化驱动源，用 PointElementReference 五个子类吸附几何创建，点→曲线→形状的链式驱动。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.4.1 点和曲线对象（p181–182）
tags: [referencepoint, curvebypoints, concept-design, pointonedge, massing, revit-api]
related_skills:
  - slug: revit-model-vs-reference-line
    relation: contrasts-with
---

# 概念设计中的 ReferencePoint 与 CurveByPoints 创建流程

## R — 原文 (Reading)

> 参照点是概念设计环境三维工作空间中指定位置的图元。ReferencePoint 可加入 ReferencePointArray 用于创建 CurveByPoints 或创建形状。点用 PointElementReference 子类创建，如 PointOnEdge、PointOnFace。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.4.1 点和曲线对象（p181–182）

---

## I — 方法论骨架 (Interpretation)

概念设计环境（体量族）的建模范式是"**先放驱动点，再长形状**"——与"画几何再标尺寸"方向相反。

- **ReferencePoint（参照点）**：三维工作空间中的定位图元，是参数化驱动源。移动它，基于它的 CurveByPoints 和 Form **跟随更新**——这是模型线不具备的特性。
- **CurveByPoints**：把一组点连成曲线。点先加入 `ReferencePointArray`，再 `NewCurveByPoints(array)` 创建。
- **点怎么吸附到几何**：用 `PointElementReference` 的五个子类：
  - `PointOnEdge` — 吸附到边（配合 `PointLocationOnCurve` 指定参数位置，可在曲线 25% 处精确放点）。
  - `PointOnFace` — 吸附到面。
  - `PointOnPlane` — 吸附到平面。
  - `PointOnEdgeEdgeIntersection` — 边边交点。
  - `PointOnEdgeFaceIntersection` — 边面交点。

**关键技巧**：曲线上定比分点用 PointOnEdge + PointLocationOnCurve 指定参数位置，**不要手工算坐标**——手工算会丢失与曲线的关联，移动曲线后点不会跟着走。

整条链在全书多处复用：放样轮廓（多条 CurveByPoints → NewLoftForm）、幕墙嵌板骨架（PointOnEdge）、自适应构件（自适应点对齐宿主点）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 放样形状
- **问题**: 概念设计中创建放样体。
- **方法论的使用**: 多条 CurveByPoints 作为轮廓 → NewLoftForm。
- **结论**: 点→曲线→形状的传递链在放样场景完整复用。
- **结果**: 放样体生成，且移动驱动点可参数化调整。

### 案例 2: 幕墙嵌板骨架
- **问题**: 在曲线特定参数处建点，搭嵌板骨架。
- **方法论的使用**: PointOnEdge 在曲线参数处创建点，构建骨架。
- **结论**: PointOnEdge 是"沿曲线定点"的标准工具。
- **结果**: 嵌板骨架正确，点与曲线保持关联。

### 案例 3: 自适应构件
- **问题**: 自适应构件需要点与宿主对齐。
- **方法论的使用**: 自适应点与宿主点对齐，点驱动放置。
- **结论**: 点驱动放置第四次复现——参照点是概念设计的通用驱动单元。
- **结果**: 构件沿点阵列自适应放置。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在体量/概念设计环境创建点、点曲线、形状。
2. 要在一条曲线 25% 处精确放一个点（定比分点）。
3. 希望形状参数化：拖动驱动点，形状跟随更新。
4. 创建幕墙嵌板、自适应构件等点驱动结构。

### 语言信号 (用户的话里出现这些就应激活)

- "概念设计环境" / "massing / conceptual design environment"
- "参照点 / ReferencePoint"
- "在曲线上 25% 处放点"（"place a point at 25% along the curve"）
- "点曲线 / CurveByPoints"
- "PointOnEdge / PointOnFace / PointOnPlane"
- "移动点形状跟着变"（"move the point and the form updates"）

### 与相邻 skill 的区分

- 与 `revit-model-vs-reference-line`：本 skill 以 ReferencePoint 点驱动，后者以模型线/参照线驱动——点与线是两种对立的驱动源。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定点吸附方式**
   - 需要吸附到 边 / 面 / 平面 / 交点？→ 选对应 PointElementReference 子类；沿边定点 → PointOnEdge + PointLocationOnCurve。
   - 完成标准: 写清每个点用哪个子类、参数位置如何给定。

2. **创建点并入数组**
   - NewReferencePoint(reference) 创建，加入 ReferencePointArray。
   - 完成标准: 点数组元素齐全，坐标/参数位置正确。

3. **创建曲线或形状并验证参数化**
   - NewCurveByPoints(array) 或 NewLoftForm 等。
   - 移动一个参照点，确认曲线/形状跟随更新。
   - 完成标准: 参数化联动生效，点-曲线-形状三层链路正确。
   - 判停条件: 若用户需要固定几何且不需要参数化，改用模型线（见 revit-model-vs-reference-line）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 项目文档（非概念设计环境）——参照点体系不存在，用普通创建 API。
- 几何固定不打算调整——用模型线更简单，参照点带来不必要的参数化负担。

### 作者在书中警告的失败模式

- 手工算坐标放点 → 点与曲线失去关联，移动曲线点不跟随。
- 在项目文档调 FamilyCreate/概念设计 API → 失败。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本概念设计 API 有扩展（如更多吸附方式），但"点→曲线→形状"链式参数化不变。

### 容易混淆的邻近方法论

- ReferencePoint（参照点）是参数化驱动源，区别于普通"坐标点"——前者有几何关联，后者没有。
- CurveByPoints 与普通 ModelCurve 是两类曲线：前者由点驱动，后者是固定草图线。

---

## 相关 skills

- **revit-model-vs-reference-line**（contrasts-with）：本 skill 以 ReferencePoint 点驱动，后者以模型线/参照线驱动——点与线是两种对立的驱动源。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
