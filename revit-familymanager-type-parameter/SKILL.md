---
name: revit-familymanager-type-parameter
description: |
  用户要给族添加类型/共享参数、用公式驱动参数、新增族类型、或改了族类型后项目实例不更新时调用。不适用于：项目文档中的项目参数/共享参数体系（那是另一套）、仅需改实例参数值。关键 trigger："给族加参数"、"add family parameter"、"共享参数怎么加到族"、"NewType"、"SetFormula 公式"、"改了族类型实例没变"。核心：FamilyManager 是族文档专属管理器；类型值须在当前类型上下文设置；修改经回载 + 显式改实例 Symbol 才影响项目。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.3.4 管理族类型和参数（p179–180）
tags: [familymanager, family-parameter, shared-parameter, formula, family-type, revit-api]
related_skills:
  - slug: revit-family-document-edit-paths
    relation: depends-on
---

# 使用 FamilyManager 管理族类型与参数

## R — 原文 (Reading)

> FamilyManager 类提供对族类型和参数的访问。使用此类可以添加和删除族类型、族和共享参数，设置不同族类型的参数值，并定义公式来驱动参数值。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.3.4 管理族类型和参数（p179–180）

---

## I — 方法论骨架 (Interpretation)

`FamilyManager` 是**族文档专属**的类型/参数管理器（`document.FamilyManager`），能力面：

- **类型管理**：`Types`（全部类型列表）、`CurrentType`（当前活动类型）、`NewType(name)`（新建类型）。
- **参数管理**：`AddParameter` 添加普通参数或**共享参数**（后者用带 SharedParameterDefinition/GUID 的重载）。
- **值设置**：`Set(param, value)` 在当前类型上下文给参数赋值。
- **公式驱动**：`SetFormula(param, formulaString)` 用公式驱动参数值。

三条生效时序铁律：

1. **类型值须在当前类型上下文设置**——先确保 CurrentType 是目标类型，再 Set。
2. **NewType 创建后自动成为当前类型**，便于连续 Set 各参数。
3. **修改经回载才影响项目**：族文档里的改动，LoadFamily 回载 + 显式改实例 `FamilyInstance.Symbol` 后，项目实例才切换。

与项目文档的参数体系区别：FamilyManager 管"族的类型参数定义"，项目文档的 ProjectParameters/共享参数管"项目/共享参数实例"——两套体系，别串。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 宽度参数创建与标注绑定
- **问题**: 给族加一个"宽度"参数并参与参数化标注。
- **方法论的使用**: AddParameter 创建"宽度"参数，与尺寸 Label 绑定。
- **结论**: AddParameter 是族参数定义的入口。
- **结果**: 参数驱动标注，改参数尺寸跟随。

### 案例 2: 新类型端到端
- **问题**: 在项目里给族新增 "2X2" 类型并让实例使用。
- **方法论的使用**: EditFamily → FamilyManager.NewType('2X2') → Set 各参数值 → LoadFamily 回载 → 改实例 Symbol。
- **结论**: NewType + Set + 回载 + 换型是完整闭环，缺一不可。
- **结果**: 新类型在项目中可用，实例正确切换。

### 案例 3: 共享参数进族
- **问题**: 给族加共享参数以便跨族/明细表统计。
- **方法论的使用**: 用带 sharedParameterDefinition/GUID 的 AddParameter 重载。
- **结论**: 共享参数走专用重载，与普通参数不同。
- **结果**: 共享参数加入族，明细表可统计。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写族工具：给族加参数（普通/共享）。
2. 用公式驱动族参数（如 面积=长×宽）。
3. 批量新增族类型并设置参数值。
4. 改了族类型，项目里的实例不更新。

### 语言信号 (用户的话里出现这些就应激活)

- "给族加参数" / "add a parameter to the family"
- "共享参数加到族里"（"shared parameter in family"）
- "用公式驱动参数"（"formula to drive parameter" / "SetFormula"）
- "新增一个族类型"（"create a new family type" / "NewType"）
- "FamilyManager"

### 与相邻 skill 的区分

- 与 `revit-family-document-edit-paths`：依赖其进入/编辑族文档的路径，再用 FamilyManager 管理类型与参数。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认族文档上下文**
   - `document.IsFamilyDocument` 为 true 才可用 FamilyManager；否则先 EditFamily/NewFamilyDocument。
   - 完成标准: 已处于族文档上下文。

2. **类型/参数操作**
   - 新增类型 → NewType(name)；加参数 → AddParameter（共享参数用带 definition 的重载）；设值 → 先确认 CurrentType 再 Set；公式 → SetFormula。
   - 完成标准: 各操作均针对 CurrentType 上下文，参数列表可查证。

3. **回载与实例生效**
   - 在项目里使用时：LoadFamily 回载 → 改目标实例 Symbol 指向新类型。
   - 完成标准: 项目实例已切换到新类型、新参数可读。
   - 判停条件: 若目标是项目文档的参数（非族参数），转项目参数体系。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 在项目文档里管项目参数/共享参数实例——另一套 API。
- 只改实例的参数值（族内已定义好的参数），无需 FamilyManager。

### 作者在书中警告的失败模式

- 不设 CurrentType 就 Set → 值落到错误类型或无效。
- 改了族类型不 LoadFamily 回载 → 项目不可见；回载后不显式改 Symbol → 实例不变。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本 FamilyManager 部分方法签名/参数类型有变化，但"类型/参数/实例三层生效时序"不变。

### 容易混淆的邻近方法论

- 族内 AddParameter（定义参数）vs 项目文档的 ProjectParameters / Binding（给类别挂参数）——定义与绑定是两件事。
- 共享参数进族用带 definition 的重载；普通参数用另一个重载——选错重载是常见错误。

---

## 相关 skills

- **revit-family-document-edit-paths**（depends-on）：依赖其进入/编辑族文档的路径，再用 FamilyManager 管理类型与参数。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
