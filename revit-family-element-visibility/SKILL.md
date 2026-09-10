---
name: revit-family-element-visibility
description: |
  用户要让族内图元只在特定视图类型（平面/3D）显示、按详细程度（粗/中/细）控制显隐、或问 FamilyElementVisibility 怎么用时调用。不适用于：视图级过滤（视图过滤器）、隐藏整个元素（Element.IsHidden）。关键 trigger："只在平面视图显示"、"3D 视图隐藏族图元"、"不同详细程度显示"、"SetVisibility"。核心：族图元可见性是"视图类型×详细程度"双轴控制，需构造 FamilyElementVisibility 对象并调 SetVisibility() 显式应用。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.3.3 族图元的可见性（p178–179）
tags: [visibility, familyelementvisibility, detail-level, view-type, family-editor, revit-api]
related_skills:
  - slug: revit-family-document-edit-paths
    relation: depends-on
---

# 族图元可见性 FamilyElementVisibility 控制

## R — 原文 (Reading)

> FamilyElementVisibility 类可用于控制族图元在项目文件中的可见性。例如，假定有一个门族，您可能只希望在该门所在项目文件的平面视图中看到可旋转门，而在三维视图中不可见。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.3.3 族图元的可见性（p178–179）

---

## I — 方法论骨架 (Interpretation)

Revit 族的图元可见性**不是开/关开关**，而是"**视图类型 × 详细程度**"的双轴控制：

- **视图类型轴**：平面视图（IsShownInPlan）、立面、3D 视图（IsShownIn3D）、剖面、明细表等。
- **详细程度轴**：Coarse（粗略）/ Medium（中等）/ Fine（精细）三级。

用法三步：
1. 构造 `FamilyElementVisibility` 对象（指定类型，如 Model 或 Annotation）。
2. 设置各轴标志，如 `IsShownInPlan = true`、`IsShownIn3D = false`。
3. 对目标族图元（ModelText、ModelCurve、三维几何等）调用 `SetVisibility(visibility)` **显式应用**——不调用则不生效。

典型效果：门族里的可旋转门体只在平面视图显示、3D 隐藏；或某图元只在"精细"级别显示。这与项目文档的视图过滤器/图形覆盖不同——后者是视图层控制，本 skill 是**族定义层**的可见性（随族走，任何项目引用都生效）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 可旋转门只在平面显示
- **问题**: 门族中可旋转门体希望平面视图可见、3D 视图不可见。
- **方法论的使用**: 创建 FamilyElementVisibility，设 IsShownInPlan=true、IsShownIn3D=false，对门体调用 SetVisibility。
- **结论**: 按视图类型控制族内图元显隐可行。
- **结果**: 平面视图正常显示可旋转门，3D 视图自动隐藏。

### 案例 2: ModelText/ModelCurve 应用可见性
- **问题**: 族编辑中文本/曲线等图元要按条件显隐。
- **方法论的使用**: 对 ModelText、ModelCurve 等多类图元应用 SetVisibility。
- **结论**: 可见性控制适用于族内多类图元，不限三维几何。
- **结果**: 族内辅助图元按需显隐。

### 案例 3: 与视图层控制的互补
- **问题**: 项目里用视图过滤器控制显隐，但希望显隐定义随族走。
- **方法论的使用**: 族定义层可见性（本 skill）与视图层过滤器（s2-f06）分层互补。
- **结论**: 两者机制不同：族内可见性影响所有引用该族的视图。
- **结果**: 明确分层，按需求选正确机制。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 族编辑中，要某构件只在平面视图显示（如符号、把手线）。
2. 要按详细程度显示不同细节（粗=简单、细=复杂）。
3. 3D 视图里某族图元要隐藏但平面视图保留。
4. 写代码设置族图元可见性，找不到正确 API。

### 语言信号 (用户的话里出现这些就应激活)

- "只在平面视图显示"（"show only in plan view"）
- "3D 视图里隐藏"（"hide in 3D"）
- "按详细程度显示"（"visibility by detail level"）
- "FamilyElementVisibility / SetVisibility"
- "族图元的可见性"

### 与相邻 skill 的区分

- 与 `revit-family-document-edit-paths`：本 skill 是进入族文档后的一个编辑动作（设可见性），依赖其进入路径。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确目标轴**
   - 用户要求的是按视图类型、按详细程度、还是两者？列出目标标志（如 IsShownInPlan=true, IsShownIn3D=false, Coarse=false）。
   - 完成标准: 写清目标维度与期望值。

2. **构造并应用**
   - `new FamilyElementVisibility()`（Model 或 Annotation）→ 设各标志 → 对目标图元 `SetVisibility(...)`，在事务中完成。
   - 完成标准: 可见性对象已创建并应用到图元，标志与目标一致。

3. **验证**
   - 切换不同视图类型/详细程度检查显隐符合预期。
   - 完成标准: 平面/3D 与各详细级别下显隐正确。
   - 判停条件: 若目标是整个元素的隐藏或视图级过滤，改走 Element.IsHidden / 视图过滤器。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要视图级过滤/图形覆盖（影响当前视图而非族定义）→ 视图过滤器。
- 隐藏整个元素而非族内子图元 → Element.IsHidden 等。

### 作者在书中警告的失败模式

- 构造了 FamilyElementVisibility 却不调用 SetVisibility → 不生效。
- 忘记事务包裹 → 修改失败。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本详细程度/视图类型标志位有扩展，双轴模型不变。

### 容易混淆的邻近方法论

- 视图可见性（View.GetVisibility / 过滤器）与族图元可见性（本 skill）作用域不同，别互换。
- FamilyElementVisibility 作用于"族内图元"，不是项目中的族实例——放错对象就不生效。

---

## 相关 skills

- **revit-family-document-edit-paths**（depends-on）：本 skill 是进入族文档后的一个编辑动作（设可见性），依赖其进入路径。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
