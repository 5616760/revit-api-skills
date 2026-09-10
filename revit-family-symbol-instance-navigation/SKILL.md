---
name: revit-family-symbol-instance-navigation
description: |
  用户要在代码里走"族↔符号↔实例"三层导航（正向创建、反向溯源、符号互换）、加载族后遍历符号建实例、或把实例换成同族另一类型时调用。不适用于：只需理解三层概念（见 revit-family-three-layer-model）、涉及系统族的访问路径（见 revit-system-vs-component-family）的场景。关键 trigger："遍历一个族的所有类型"、"换门的型号"、"instance.Symbol = "、"加载 rfa 后怎么创建实例"、"拿到实例所属的族"。注意体量/内建构件族名为空，遍历时需特判。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2 族实例（p158–159）
tags: [family, familysymbol, navigation, symbol-swap, revit-api]
related_skills:
  - slug: revit-family-three-layer-model
    relation: depends-on
  - slug: revit-system-vs-component-family
    relation: depends-on
---

# 族 Family、族符号 FamilySymbol、族实例 FamilyInstance 三元关系导航

## R — 原文 (Reading)

> Revit 平台 API 中的族是由三个对象表示：族、族符号、族实例。族对象表示整个族……族对象包含多个 FamilySymbols，用于获取所有的族符号，以便符号之间的实例互换。族符号对象表示对应于 Revit 用户界面中某个类型的一组特定的属性设置。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2 族实例（p158–159）

---

## I — 方法论骨架 (Interpretation)

这是把"族体系"落地成可执行代码的导航地图。三条通道记住就够：

1. **正向通道（创建）**：加载族文件 → 从 `document.FamilySymbols`（或 `family.Symbols`）中按名字筛出目标符号 → `NewFamilyInstance` 用符号创建实例。创建桌子、梁、门三个案例都复用这条链路。
2. **反向通道（溯源）**：`instance.Symbol` 得类型 → `symbol.Family` 得族 → 必要时用 `family.OwnerFamily` 进入族文档。实例→符号→族→族文档的反向路径在"在项目里编辑族"时反复用到。
3. **互换通道（改型）**：直接给 `FamilyInstance.Symbol` 赋新符号即可换类型，不需要删除重建。

例外要常备：注释图元走平行封装 `AnnotationSymbolType`/`AnnotationSymbol`；体量、内建构件在 UI 中"族/类型"字段为空，按三元关系强取会 NullReference。系统族（墙等）不经此三元体系，走专用类型类。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 桌子 / 梁 / 门创建的导航路径
- **问题**: 三个不同对象，创建流程能否统一？
- **方法论的使用**: 每个案例都走"加载族 → 遍历 FamilySymbols 取符号 → NewFamilyInstance"，正向通道复用。
- **结论**: 三元导航骨架与具体构件类别无关。
- **结果**: 桌子、梁、门创建代码结构完全一致，仅参数不同。

### 案例 2: 在族编辑场景反向溯源
- **问题**: 项目中拿到一个实例，想知道它的族并打开族文档修改。
- **方法论的使用**: 走反向通道 实例→符号→族→族文档（EditFamily / OwnerFamily）。
- **结论**: 反向通道是"项目上下文↔族文档上下文"的桥。
- **结果**: 成功在族编辑场景定位到内嵌符号（s2b-c06/c09）。

### 案例 3: 实例换型号
- **问题**: 想把一扇门换成同族另一尺寸，删除重建太浪费。
- **方法论的使用**: 互换通道——改 `FamilyInstance.Symbol`。
- **结论**: 类型是引用不是几何。
- **结果**: 实例即时切换类型，几何跟随更新。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 加载了一个 .rfa，要写代码在项目里放它的实例——先遍历符号。
2. 已有实例，要拿到它属于哪个族、哪个类型。
3. 想把一批实例换成同族的不同类型（改型号/改尺寸）。
4. 遍历全项目族实例，要按族/类型分组统计。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么遍历一个族的所有类型/符号"（"iterate family symbols"）
- "怎么把门换成另一型号"（"change the type of this instance" / "switch symbol"）
- "instance.Symbol = xxx" 这种赋值写法
- "拿到实例的族"（"get family of instance"）
- "加载了族文件，怎么创建实例"

### 与相邻 skill 的区分

- 与 `revit-family-three-layer-model`：依赖其三层概念模型；后者只回答"三层是什么"，本 skill 负责代码中如何在三层间导航。
- 与 `revit-system-vs-component-family`：依赖其系统族/构件族判别；本 skill 的导航路径仅对构件族成立。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定导航方向**
   - 创建 → 正向；溯源 → 反向；换型 → 互换。
   - 完成标准: 写清目标（要实例 / 要符号 / 要族）。

2. **执行导航并校验空值**
   - 反向: `instance.Symbol`（判 null，体量/内建可能为空）→ `symbol.Family`。
   - 正向: `doc.FamilySymbols` 按 `Name` 精确匹配筛选，找不到则先 LoadFamily。
   - 互换: 先确认目标符号属于同一族，再赋 `instance.Symbol`。
   - 完成标准: 导航每一步都有 null 检查，拿到目标对象。
   - 判停条件: 若实例的族名以 "System Family" 开头或为 null，则跳 `revit-system-vs-component-family`。

3. **收尾验证**
   - 换型后检查实例参数/几何已更新；创建后确认实例已存在于文档。
   - 完成标准: 在事务提交后能通过反向通道取回同一对象。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需按 Category 过滤图元，不关心具体族类型。
- 图元是系统族（墙/楼板/屋顶）——它们没有 FamilySymbol 三元对象，走 WallType 等专用类。

### 作者在书中警告的失败模式

- 体量、内建构件族名字段为空——按三元关系取 Symbol/Family 直接 NullReference。
- 加载族后不刷新符号集合，符号还没载入就取会拿不到。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：新版中类型系统演进（Element.GetTypeId 等），但 Family/Symbol/Instance 三元关系仍是核心骨架。

### 容易混淆的邻近方法论

- AnnotationSymbolType/AnnotationSymbol 是注释图元的平行三元体系，别用 Family 系方法去取。
- Category（类别）维度和族维度正交，别混用。

---

## 相关 skills

- **revit-family-three-layer-model**（depends-on）：依赖其三层概念模型；后者只回答"三层是什么"，本 skill 负责代码中如何在三层间导航。
- **revit-system-vs-component-family**（depends-on）：依赖其系统族/构件族判别；本 skill 的导航路径仅对构件族成立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
