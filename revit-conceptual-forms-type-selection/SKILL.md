---
name: revit-conceptual-forms-type-selection
description: |
  用户要在体量/概念设计环境创建拉伸、旋转、放样、融合、表面形状，或困惑"体量里用 NewExtrudeForms 还是族文件的 NewExtrusion"时调用。不适用于：族文件环境创建三维形状（用具体类，见 revit-familyitemfactory-shape-creation）。关键 trigger："体量族里创建旋转体/放样"、"create form in massing"、"Form 类"、"表面形状"、"没有厚度的面片"。核心：概念设计所有形状统一用 Form 类（New*Forms 系列），与族文件五具体类双轨割裂，不能混用。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.4.2 形状（p184–185）
tags: [conceptual-design, massing, form, loft, surface-form, revit-api]
related_skills:
  - slug: revit-referencepoint-curvebypoints
    relation: composes-with
  - slug: revit-model-vs-reference-line
    relation: composes-with
---

# 概念设计环境中 Forms 形状创建类型选择

## R — 原文 (Reading)

> 概念设计环境提供了创建新形状的功能。可以创建以下类型的形状：拉伸、旋转、放样、放样融合、旋转和表面形状。不同于族创建中所使用的 Blend、Extrusion、Revolution、Sweep 和 SweptBlend 类，体量族对所有类型的形状都使用 Form 类。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.4.2 形状（p184–185）

---

## I — 方法论骨架 (Interpretation)

概念设计（体量）环境能创建六类形状：拉伸、旋转、放样、放样融合、融合、表面形状。**最重要的规则是双类体系割裂**：

- **族文件环境**：用五个具体类——Blend、Extrusion、Revolution、Sweep、SweptBlend。
- **概念设计/体量环境**：**所有形状统一用 Form 类**，方法为 New*Forms 系列——NewExtrudeForms（拉伸）、NewRevolveForms（旋转）、NewSweepForms（放样）、NewLoftForm（放样融合）等。

两套类体系**不能混用**：在体量里用族文件的 NewExtrusion 是错的；反之亦然。这是 Revit API 历史演进留下的双轨。

几个要点：
- **表面形状（Surface Form）**：由**单个轮廓**创建、类似拉伸但**不给定高度**——形成"面"而非实体。要一个没有厚度的面片，选"表面形状"而不是"拉伸+零深度"。
- 创建入口：经 `document.FamilyCreate`（与族文件同一工厂），提供轮廓（CurveArrArray），事务中创建。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 放样实操
- **问题**: 概念设计中用多条轮廓曲线创建放样融合形状。
- **方法论的使用**: 多条 CurveByPoints 轮廓 → NewLoftForm。
- **结论**: 放样在概念设计走 Form 类方法。
- **结果**: 放样形状生成且受驱动曲线控制。

### 案例 2: 双类体系的对照
- **问题**: 体量里误用族文件的形状类。
- **方法论的使用**: 确认体量环境统一用 Form 类（NewExtrudeForms 等），族文件用具体类。
- **结论**: 同一"拉伸"在两环境有不同类与不同方法名。
- **结果**: 按环境选类，避免编译/运行错误。

### 案例 3: 表面形状的语义
- **问题**: 需要一个没有厚度的面片（如曲面嵌板）。
- **方法论的使用**: 用表面形状——单个轮廓、不给高度。
- **结论**: 表面形状 = 不给定高度的拉伸，是"面"不是实体。
- **结果**: 面片创建成功，无厚度。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在体量族里创建旋转体、放样、融合等形状。
2. 分不清"体量里用什么类 vs 族文件里用什么类"。
3. 要创建表面形状（无厚度面片）。
4. 写概念设计自动化工具，需要按形状类型选方法。

### 语言信号 (用户的话里出现这些就应激活)

- "体量族里创建旋转/放样"（"create revolve/loft in massing"）
- "NewExtrudeForms / NewLoftForm"
- "Form 类"（"the Form class"）
- "表面形状"（"surface form"）
- "没有厚度的面片"

### 与相邻 skill 的区分

- 与 `revit-referencepoint-curvebypoints`：组合其点/曲线输入来创建 Form 形状，输入在前、形状输出在后。
- 与 `revit-model-vs-reference-line`：组合其线驱动输入来创建 Form 形状，输入在前、形状输出在后。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断文档环境**
   - 体量/概念设计 → Form 类；族文件 → 具体类。
   - 完成标准: 明确环境并写出对应类体系名称。

2. **按形状类型选方法**
   - 拉伸 → NewExtrudeForms；旋转 → NewRevolveForms；放样 → NewSweepForms；放样融合 → NewLoftForm；面片 → 表面形状（单轮廓、不给高度）。
   - 完成标准: 方法与形状类型一一对应。

3. **构造输入并创建**
   - 轮廓用 CurveArrArray（闭合），事务中经 document.FamilyCreate 调用。
   - 完成标准: 形状创建成功、类型正确。
   - 判停条件: 若在族文件（非体量）环境，转 `revit-familyitemfactory-shape-creation`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 族文件编辑环境——用具体类（Blend/Extrusion/Revolution/Sweep/SweptBlend）。
- 需要实体且有确定厚度的拉伸——那是普通拉伸，表面形状仅用于"面"语义。

### 作者在书中警告的失败模式

- 混用两套类体系 → 编译/运行错误。
- 轮廓不闭合或 CurveArrArray 结构错误 → 创建失败。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：Form 类方法与概念设计 API 在后续版本有演进（部分方法废弃/改名），双轨思想延续。

### 容易混淆的邻近方法论

- 族文件 NewExtrusion 与概念设计 NewExtrudeForms 名字相近但不同类——按环境区分。
- 表面形状与"拉伸+零深度"不是一回事：前者是面，后者是退化实体，语义不同。

---

## 相关 skills

- **revit-referencepoint-curvebypoints**（composes-with）：组合其点/曲线输入来创建 Form 形状，输入在前、形状输出在后。
- **revit-model-vs-reference-line**（composes-with）：组合其线驱动输入来创建 Form 形状，输入在前、形状输出在后。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
