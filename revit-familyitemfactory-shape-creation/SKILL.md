---
name: revit-familyitemfactory-shape-creation
description: |
  用户在族文件中创建拉伸/放样等三维形状、项目文档调 FamilyCreate 报错、或困惑 document.Create 与 FamilyCreate 区别时调用。不适用于：项目文档创建普通图元（用 document.Create）、概念设计环境创建形状（走 Form 类）。关键 trigger："在族里创建拉伸"、"FamilyCreate 报错"、"create extrusion"、"轮廓传什么类型"。核心：FamilyItemFactory 仅族文档可用（经 FamilyCreate），项目文档走 Create——同源 ItemFactoryBase，按文档类型分工。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.3.2 在族中创建图元（p174–175）
tags: [familyitemfactory, familycreate, extrusion, sweep, family-editor, revit-api]
related_skills:
  - slug: revit-family-document-edit-paths
    relation: depends-on
  - slug: revit-conceptual-forms-type-selection
    relation: contrasts-with
---

# 在族文件中使用 FamilyItemFactory 创建三维形状图元

## R — 原文 (Reading)

> FamilyItemFactory 类提供了在族文件中创建图元的能力。它通过 Document.FamilyCreate 属性进行访问。FamilyItemFactory 是从 ItemFactoryBase 类派生的，它是一个在 Revit 项目文件和族文件中创建图元的实用程序。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.3.2 在族中创建图元（p174–175）

---

## I — 方法论骨架 (Interpretation)

Revit 的"创建 API"按文档类型分裂成**两个工厂**，但同源于 `ItemFactoryBase`：

- **项目工厂**：`document.Create` —— 项目文档里建普通图元。
- **族工厂**：`document.FamilyCreate`，返回 `FamilyItemFactory` —— **只能在族文档用**（`IsFamilyDocument == true`）。在项目文档里调它直接失败。

族工厂能干什么：
- 创建三维形状：`NewExtrusion`（拉伸）、`NewSweep`（放样）等。
- 创建草图/参照图元、概念设计的点/线/形状（NewReferencePoint、NewLoftForm 等）——**同一个工厂在概念设计语境大规模复用**。
- 所有创建必须**在事务中**进行，且轮廓要**闭合**。

结构约定：轮廓参数是 `CurveArrArray`——**数组套数组**（外层一条轮廓组，内层闭合环）。这是族草图体系特有的传参结构，传错类型编译不过，传不闭合的线创建失败。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 族中拉伸与放样
- **问题**: 在族文件里要创建拉伸体与放样体。
- **方法论的使用**: 走 FamilyItemFactory 的 NewExtrusion / NewSweep，事务内提供闭合轮廓。
- **结论**: 族工厂是族内三维形状的唯一入口。
- **结果**: 形状创建成功并受族参数驱动。

### 案例 2: 概念设计环境复用同一工厂
- **问题**: 概念设计（体量）环境里创建参照点、点曲线、放样形状。
- **方法论的使用**: 全部经 document.FamilyCreate（NewReferencePoint / NewCurveByPoints / NewLoftForm）。
- **结论**: FamilyItemFactory 不只管"族"，概念设计也是它的地盘。
- **结果**: 点→线→形状的创建链路统一走族工厂。

### 案例 3: 分割表面
- **问题**: 概念设计中创建分割表面。
- **方法论的使用**: NewDividedSurface 同样经 FamilyCreate。
- **结论**: 第三次复现"概念设计=族工厂"规律。
- **结果**: 分割表面创建成功。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写族编辑工具，要在族文件里创建三维形状。
2. 在项目文档里调 document.FamilyCreate 报错，怀疑是上下文问题。
3. 创建拉伸/放样时不确定轮廓参数类型。
4. 在概念设计环境中创建点/线/形状。

### 语言信号 (用户的话里出现这些就应激活)

- "在族里创建拉伸/放样"（"create extrusion in family file"）
- "FamilyCreate 报错"（"FamilyCreate not available"）
- "document.Create 和 FamilyCreate 的区别"
- "CurveArrArray / 闭合轮廓"
- "NewExtrusion / NewSweep"

### 与相邻 skill 的区分

- 与 `revit-family-document-edit-paths`：依赖其进入族文档的路径，再调用 FamilyCreate 工厂创建几何。
- 与 `revit-conceptual-forms-type-selection`：本 skill 用族文件环境的工厂与具体形状类，后者用概念设计环境的 Form 类体系——同是创建形状但环境对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认上下文**
   - 检查 `document.IsFamilyDocument`。为 true → 用 `document.FamilyCreate`；为 false（项目文档）→ 改用 `document.Create` 或提示语境不符。
   - 完成标准: 明确写出当前文档类型与对应工厂。

2. **准备轮廓与事务**
   - 构造闭合的 `CurveArrArray`（外层数组=轮廓组，内层=闭合环）。
   - 启动事务。
   - 完成标准: 轮廓闭合、类型正确、事务已开启。

3. **调用创建方法**
   - `familyCreate.NewExtrusion(...)` / `NewSweep(...)` / 概念设计方法等；事务提交。
   - 完成标准: 形状创建成功，事务正常提交。
   - 判停条件: 若在概念设计（体量）环境且要创建 Form 形状，转到 `revit-conceptual-forms-type-selection`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 项目文档中创建普通图元——用 document.Create。
- 概念设计环境创建 Form 形状——虽然入口相同，但形状类选择规则不同（见 Forms skill）。

### 作者在书中警告的失败模式

- 在项目文档调 FamilyCreate → 直接失败。
- 轮廓不闭合或 CurveArrArray 结构错误 → 创建失败/异常。
- 所有创建必须在事务中，否则报"无事务"错误。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本族编辑器中三维创建 API 有演进（如 SketchPlane 要求更严），但双工厂架构延续。

### 容易混淆的邻近方法论

- 族文件的 `NewExtrusion`（具体类）与概念设计的 `NewExtrudeForms`（Form 类）名字相近但归属不同类体系——两套不能混用（详见 Forms skill）。
- FamilyItemFactory 只负责"创建图元"，与 FamilyManager（参数管理）职责不同。

---

## 相关 skills

- **revit-family-document-edit-paths**（depends-on）：依赖其进入族文档的路径，再调用 FamilyCreate 工厂创建几何。
- **revit-conceptual-forms-type-selection**（contrasts-with）：本 skill 用族文件环境的工厂与具体形状类，后者用概念设计环境的 Form 类体系——同是创建形状但环境对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
