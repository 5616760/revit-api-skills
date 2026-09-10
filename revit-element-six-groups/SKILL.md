---
name: revit-element-six-groups
description: |
  按功能六组预判 Revit 图元 API：Model、Sketch、View、Group、Annotation（尺寸/标记/文字）、Information。各组 API 不同：Model 用 FamilyInstance/Create，Annotation 用 NewDetailCurve/NewTag——先定组再选 API。信号："注释类图元 / annotation elements"、"创建尺寸/标记 / create dimension tag"。不适用：类型-实例维度。
  Trigger: annotation / model elements。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.5.1（约p067）
tags: [revit-api, element-classification, annotation, model-elements, revit-concepts]
related_skills:
  - slug: revit-filter-quick-slow-logical
    relation: composes-with
  - slug: revit-category-family-symbol-instance
    relation: composes-with
---

# 图元分类六组及其使用场景

## R — 原文 (Reading)

> Revit 图元分为六组：Model（模型）、Sketch（草图）、View（视图）、Group（成组）、Annotation（注释）和 Information（信息）。每一组都包含相关的图元和其对应的符号。
>
> — 宦国胜, 第1章 1.5.1（约p067）

---

## I — 方法论骨架 (Interpretation)

动手操作任何图元前，先判断它属于六组中的哪一组，因为**组决定 API 行为**：

- **Model（模型）**：建筑里的物理项。细分三层——族实例（FamilyInstance）、主体图元（墙、楼板、屋顶、天花板等，有"承载"语义）、结构图元。创建走 NewFamilyInstance 或各主体的 Create 方法。
- **Sketch（草图）**：绘图时定义轮廓/路径的临时几何，编辑语义与模型图元完全不同。
- **View（视图）**：平面/立面/三维视图本身也是图元，创建走 ViewPlan.Create 一族。
- **Group（成组）**：多图元的打包单元，编辑要考虑组内成员联动。
- **Annotation（注释）**：尺寸标注、标记、文字注释等图纸信息——**只能存在于视图中**，创建走视图专有 API（NewDetailCurve、NewTag 等），不能像模型图元那样直接放。
- **Information（信息）**：非图形的信息类图元。

这张表的用途是事前预判：拿到"要创建/编辑 XX"的需求，先定位组，就能预知该用哪族 API、有哪些天然限制（如注释绑定视图、草图随主体）。跨组套用 API 是新手报错的高发原因。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 过滤注释类图元
- **问题**: 需要过滤出所有注释类图元（尺寸、标记、文字）。
- **方法论的使用**: 定位到 Annotation 组——Dimension、IndependentTag、TextNote、AnnotationSymbol 都在其中，按这一组组织过滤条件。
- **结论**: "注释类"不是杂项，是功能六组中有明确成员的一组。
- **结果**: 过滤条件按组成员设计，覆盖完整。

### 案例 2: 创建方式差异
- **问题**: 分别创建模型图元和注释图元时 API 报错/行为怪异。
- **方法论的使用**: 依据组差异选 API——Model 组走 FamilyInstance/Create 方法；Annotation 组走视图专有的 NewDetailCurve/NewTag；View 组走 ViewPlan.Create。
- **结论**: 创建 API 按组分流，不能跨组套用。
- **结果**: 各组用各自入口后创建正常。

### 案例 3: 模型组内部细分
- **问题**: 对墙/楼板/屋顶的处理与对族实例的处理逻辑不同。
- **方法论的使用**: 依据模型组三层细分——主体图元有承载语义，族实例是放置产物，分别处理。
- **结果**: 主体与实例的编辑路径各归其位。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要创建某类图元，不知道该用哪个 API 家族（Create? NewFamilyInstance? NewTag?）。
2. 写过滤条件，需要按"模型/注释/视图/信息"粗分类组织。
3. 注释图元操作报错，怀疑 API 用错组（如把注释当模型图元创建）。

### 语言信号 (用户的话里出现这些就应激活)

- "图元分几类 / element classification / types of elements"
- "注释类图元 / annotation elements / dimensions and tags"
- "创建尺寸/标记 / create dimension / new tag"
- "模型图元 / model elements / host elements"

### 与相邻 skill 的区分

- 与 `revit-category-family-symbol-instance` 的区别: 六组是**功能维度**分类（这东西干什么用）；Category→Family→Symbol→Instance 是**类型-实例维度**分类（这东西怎么定义与实例化）。两套体系正交，先功能后类型。
- 与 `revit-filter-quick-slow-logical` 的区别: 本 skill 帮你把目标翻译成类别/组成员；过滤器组合策略是下一步的工程实现。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **给目标图元定组**：判断属于 Model/Sketch/View/Group/Annotation/Information 哪一组（Model 内再分实例/主体/结构）。
   - 完成标准: 组别明确写出，Model 组需写明细分层。

2. **按组选 API 家族**：Model → Create/NewFamilyInstance；Annotation → 视图专有 API；View → ViewPlan.Create 等；Sketch/Group/Information → 各自专属入口。
   - 完成标准: API 选择与组别对应，无跨组借用。
   - 判停条件: 若是注释图元需求，先确认目标视图存在——注释不能脱离视图创建。

3. **核对组内约束**：检查该组的天然限制（注释绑视图、主体有承载关系、成组有联动）是否已在代码中处理。
   - 完成标准: 每条组级约束有对应处理或显式说明。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 讨论某图元的类型/实例层次导航（Symbol→Family）——转第二维度分类 skill。
- 具体过滤器的性能设计——本 skill 只管"目标是什么"，不管"怎么高效找"。

### 作者在书中警告的失败模式

- 把注释图元当模型图元创建——注释必须依附视图，用模型 API 直接失败。
- 忽视模型组内部主体/实例的差异，用同一套逻辑处理墙与族实例。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：六组框架沿用至今，但新版图元种类更多（如分析模型、路径钢筋等），个别归属需查最新文档；组间 API 形态也有演进（如 NewTag 等被归入更统一的创建入口趋势）。

### 容易混淆的邻近方法论

- "六组功能分类"与"Category 类别"不是一回事：Category 是 API 里的显式枚举/对象（OST_Walls 等），六组是概念分组——过滤时用 Category，预判行为时用六组。

---

## 相关 skills

- revit-filter-quick-slow-logical：composes-with——六组分类把目标翻译成类别/组成员，过滤器组合策略是下一步的工程实现。
- revit-category-family-symbol-instance：composes-with——六组是功能维度、四层模型是类型-实例维度，两套分类正交，配合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
