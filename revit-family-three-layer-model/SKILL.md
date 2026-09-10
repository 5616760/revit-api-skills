---
name: revit-family-three-layer-model
description: |
  用户困惑"族、族符号、族实例到底什么关系"、想知道 Family 与 FamilySymbol 的包含/从属结构、需要从实例反向拿到类型或族、或创建实例前不知符号从哪来时调用。不适用于：只需按 Category 过滤图元、明确知道目标类型对象（如直接 WallType）的场景。关键 trigger："族和类型的区别"、"family vs family symbol"、"get the family of this instance"、"怎么新建某个类型的梁"、"改类型为什么实例没变"。本 skill 讲三层是什么（概念模型），代码导航操作见 revit-family-symbol-instance-navigation。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.5.2（p071–072）
tags: [family, familysymbol, familyinstance, object-model, revit-api]
related_skills:
  - slug: revit-system-vs-component-family
    relation: composes-with
---

# 族与类型的关系：Family / FamilySymbol / FamilyInstance 三层结构

## R — 原文 (Reading)

> Revit 平台 API 中的族有三个对象：Family（族）、FamilySymbol（族符号）、FamilyInstance（族实例）。Family 对象表示整个族；FamilySymbol 表示一组特定的族设置；FamilyInstance 是 FamilySymbol 的实例。每个族实例都有族符号。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第1章 1.5.2（p071–072）

---

## I — 方法论骨架 (Interpretation)

把 Revit 里"族"的概念拆成三个递进的对象层，类似"文件 → 款式 → 实物"：

- **Family（族）** = 一整套构件的定义，如"梁"这个族。它像一个容器，里面装着这个族的所有类型。
- **FamilySymbol（族符号）** = 族内部的一组具体参数设置，对应 UI 里的"类型"下拉项，如"混凝土矩形梁：16×32"。
- **FamilyInstance（族实例）** = 被放进项目里的单个具体构件，身上带着它属于哪个符号。

关键点在于：这不是面向对象的类继承关系，而是**包含/从属关系**——一个族包含一到多个符号，一个符号对应零到多个实例，实例永远只归属一个符号。写代码时最常见的路径是从实例反向导航：`instance.Symbol` 拿到类型，`symbol.Family` 拿到族。系统族（如墙）虽然不能加载 .rfa，但也遵守"族有类型、类型有实例"的同一逻辑，只是走 WallType 等专用类。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建桌子 / 梁 / 门前先走完三层
- **问题**: 要在项目里放一张桌子，只知道有某个族文件，不知道实例对象从哪来。
- **方法论的使用**: 书中的创建案例（桌子、梁、门）统一走"加载族 → 遍历其 FamilySymbols → 用符号创建 FamilyInstance"的链路，三层逐级落地。
- **结论**: 实例不能凭空 new，必须从"族→符号→实例"逐层取得。
- **结果**: 创建流程稳定可复用，四个对象案例（桌子/梁/门/床）共享同一导航骨架。

### 案例 2: 实例互换类型
- **问题**: 想换个门的类型又不想删除重画。
- **方法论的使用**: 三层关系中符号是"可换装"的一层——直接替换 `FamilyInstance.Symbol` 指向的符号即可。
- **结论**: 类型互换是"改引用"，不是"改几何"。
- **结果**: 实例类型即时切换，无需重建。

### 案例 3: 系统族也套三层模型
- **问题**: 系统族（墙/楼板）没有族文件，三层关系是否失效？
- **方法论的使用**: 系统族同样有"族、类型、实例"概念，但类型经 Document 内置类型（WallType 等）访问，不走 Family/FamilySymbol 对象。
- **结论**: 三层模型是普适骨架，系统族只是 API 入口不同。
- **结果**: 用 UI 族名是否以 "System Family" 开头即可区分两类族。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 学习/面试场景：问"族、族符号、族实例有什么区别和联系"。
2. 拿到一个 Element，想知道它属于哪个族、哪个类型。
3. 写创建代码，知道要放某类构件但不知道 FamilySymbol 从哪获取。
4. 改了类型却发现实例外观没变，怀疑自己搞错了层。

### 语言信号 (用户的话里出现这些就应激活)

- "族、族符号、族实例是什么关系"
- "Family、FamilySymbol、FamilyInstance 的区别"
- "get the family / type of an instance"（"拿到实例的族/类型"）
- "怎么创建某个类型的族实例"
- "instance.Symbol 是什么" / "symbol.Family 是什么"

### 与相邻 skill 的区分

- 与 `revit-system-vs-component-family`：本 skill 是族三层模型的认知基础，后者在三层之上区分系统族/构件族两类，组合起来完整理解族体系。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位对象所在层**
   - 完成标准: 通过 `obj.GetType()` / `typeOf(obj)` 确认拿到的是 Family、FamilySymbol 还是 FamilyInstance（或 ElementType）。

2. **确定需要的导航方向**
   - 反向（实例→族）：`instance.Symbol` → `symbol.Family`。
   - 正向（族→类型）：`family.Symbols` / `document.FamilySymbols` 遍历筛选。
   - 完成标准: 明确写出本次目标对象是哪一层。

3. **执行导航并校验边界**
   - 访问 `Symbol` / `Family` 前先判空；体量、内建构件的符号可能为 null。
   - 需要改类型时直接赋新符号，不重建实例。
   - 完成标准: 拿到目标对象（或确认其为空并给出原因）。
   - 判停条件: 若对象是系统族（族名 "System Family" 前缀），则跳到 `revit-system-vs-component-family` skill 处理。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只是按 Category 过滤一批图元，不需要关心具体族/类型归属。
- 已经持有类型对象（如直接通过 WallType）操作，三层导航反而绕路。

### 作者在书中警告的失败模式

- 体量和内建构件没有族名/符号，按"三层一定有值"假设遍历会遭遇 null。
- 系统族不可经 Family 体系访问——对系统族调族加载/编辑路径会失败。

### 作者的盲点 / 时代局限

- 基于 Revit 2014 / .NET 4.0：部分导航方法（如 Element.GetTypeId() 及类型体系）在后续版本中位置有调整；三层对象模型的命名延续至今但方法面变化不小。

### 容易混淆的邻近方法论

- Category（类别）是**另一维度**（revit-category-family-symbol-instance 的四层分类含 Category），Family 不是 Category 的别名——一个族归一个类别，但类别划分更粗。
- FamilySymbol 是 ElementType 的子类，等价于"类型"概念，与项目文档里的通用类型体系（如 WallType）是同一抽象的两个入口。

---

## 相关 skills

- **revit-system-vs-component-family**（composes-with）：本 skill 是族三层模型的认知基础，后者在三层之上区分系统族/构件族两类，组合起来完整理解族体系。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
