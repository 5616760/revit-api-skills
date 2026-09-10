---
name: revit-category-family-symbol-instance
description: |
  用类型-实例四层模型导航 Revit 图元：Category→Family→FamilySymbol（"族类型"）→FamilyInstance。导航链：FamilyInstance.Symbol→FamilySymbol→Family，Element.Category 取类别；创建实例前先 Symbol.Activate()。信号："族类型 / family type"、"从实例找族 / instance to family"、"symbol vs instance"。不适用：功能六组分类。
  Trigger: Category / Family / FamilySymbol / Symbol.Activate。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.5.2（约p069-072）
tags: [revit-api, family, category, object-model, revit-concepts]
related_skills: []
---

# 图元的第二维度分类：Category / Family / Symbol / Instance

## R — 原文 (Reading)

> 图元也可分为以下几类：Category、Family、Symbol、Instance。
> 族是某类别中图元的类；Revit 平台 API 中的族有三个对象：Family、FamilySymbol、FamilyInstance。
> 符号通常用于定义实例的不可见图元。
> 实例是在建筑或绘图图纸中有具体位置的项目。
>
> — 宦国胜, 第1章 1.5.2（约p069-072）

---

## I — 方法论骨架 (Interpretation)

除了功能六组，Revit 图元还有一套**类型-实例四层**分类，这四层回答"这东西是怎么定义出来的"：

- **Category（类别）**：最大粒度的归类（墙、门、窗……）。API 里从 `Element.Category` 读；全项目类别集合在 `Document.Settings.Categories`。
- **Family（族）**：某类别下的"类文件"，定义几何/参数模板。
- **FamilySymbol（符号）**：族的一个具体"类型"——界面里叫"族类型"，是**不可见的定义**；实例按它生成。API 里创建实例前必须先 `Symbol.Activate()`。
- **FamilyInstance（实例）**：按 Symbol 放置到项目里的具体对象，有位置、有实例参数。

导航链条是固定套路：
- 从实例向上：`FamilyInstance.Symbol`（拿到类型）→ `FamilySymbol.Family`（拿到族）→ `Family`/`Element` 的 `.Category`（拿到类别）。
- 从类别向下：Categories → 过滤出 Family（FamilySymbolFilter）→ 过滤出 Symbol → 用 Symbol 创建 Instance。

记住一句换算：**界面术语"族类型" = API 的 FamilySymbol**；界面说"实例属性/类型属性"分别对应 Instance 与 Symbol 上的参数。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 从实例导航到族
- **问题**: 手里有一个 FamilyInstance，要拿到它的 Family。
- **方法论的使用**: 沿固定链 `FamilyInstance.Symbol` → `FamilySymbol.Family`；类别用 `Element.Category`。
- **结论**: 四层之间的导航路径是唯一的、可背的。
- **结果**: 无需猜测 API，两跳到达 Family。

### 案例 2: 获取类别全集
- **问题**: 第1章示例需要枚举项目中的类别做处理。
- **方法论的使用**: `Document.Settings.Categories` 是包含所有类别对象的映射，从它入手。
- **结论**: 类别不是硬编码枚举的，从文档设置域取。
- **结果**: 拿到与项目一致的类别全集。

### 案例 3: 系统族与构件族
- **问题**: 墙这类系统族没有独立族文件，与门/窗构件族的导航方式混淆。
- **方法论的使用**: 依据第二维度的分层（系统族的 Symbol/Family 挂在文档类型集合上，构件族走 LoadFamily 路径），分路径处理。
- **结论**: 四层模型对两者都成立，但入口不同。
- **结果**: 两类族的类型获取各自走通。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 代码里持有实例/类型/族之一，需要向上或向下导航到另一层。
2. 创建族实例前不知道要先 Activate 对应 Symbol。
3. 界面术语（族类型）与 API 术语（FamilySymbol）对不上，找不到类。
4. 按类别/族/类型逐层过滤或统计。

### 语言信号 (用户的话里出现这些就应激活)

- "族类型 / family type / FamilySymbol"
- "实例找到族 / from instance to family / get symbol"
- "类别 / category / Settings.Categories"
- "创建族实例 / place family instance / NewFamilyInstance"

### 与相邻 skill 的区分

- 与 `revit-element-six-groups` 的区别: 六组管"功能语义"（注释必须绑视图这类行为约束）；四层管"定义层次"（类型与实例的关系）。两套正交：一个族实例同时属于 Model 组和四层模型的底层。
- 与 `revit-document-function-map` 的区别: Categories/FloorTypes 等入口是 Document 功能地图里的"设置/类型集合"域；本 skill 讲拿到之后四层之间怎么走。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位当前层**：手头对象是 Category / Family / FamilySymbol / FamilyInstance 哪一层？
   - 完成标准: 层别明确（注意界面的"类型属性"即 Symbol 层）。

2. **沿链导航**：向上用 Instance.Symbol → Symbol.Family → .Category；向下用 Categories/FamilySymbolFilter 逐层过滤。
   - 完成标准: 导航代码只沿四层链走，无越层硬猜属性。
   - 判停条件: 若目标就是创建实例 → 跳步骤 3。

3. **创建实例的规范**：取到 FamilySymbol 后先 `Symbol.Activate()`（如未激活），再 NewFamilyInstance 放置。
   - 完成标准: 放置调用前 Activate 已处理。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 讨论图元功能行为约束（注释绑视图等）——转六组分类。
- 系统族的类型编辑细节（墙类型参数结构）——属墙体/类型参数专属话题。

### 作者在书中警告的失败模式

- 混用界面与 API 术语：把"族类型"当 Family 处理——实际是 FamilySymbol。
- 创建实例前忘记 Activate Symbol——放置失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：四层模型稳定，但新版对 Activate 的要求有放宽（部分场景自动激活）、NewFamilyInstance 重载更丰富；直墙等系统族仍无独立 Family 文件，导航方式与书中描述一致。

### 容易混淆的邻近方法论

- FamilySymbol（类型定义，改它影响所有后续实例）与 FamilyInstance（实例，改它只影响自己）上的参数读写是两条路径——"改参数改错了层"是常见事故（想改一个门的结果全楼的门都变了）。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
