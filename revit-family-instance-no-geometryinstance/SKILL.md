---
name: revit-family-instance-no-geometryinstance
description: |
  遍历族实例几何时顶层拿到 Solid 而非 GeometryInstance、导致漏几何——不是 bug：局部连接/相交/实例放置触发唯一副本生成，GeometryInstance 消失、Solid 浮到顶层。必须对顶层同时做 GeometryInstance 与 Solid/Curve/Mesh 类型分派。触发："为什么拿到的是 Solid"、"geometry instance missing"、"遍历梁几何结果时有时无"。不适用于：只需嵌套族展开的窄场景。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.2（约 p218）
tags: [geometryinstance, counterexample, type-dispatch, geometry-extraction, revit-internals]
related_skills:
  - slug: revit-element-geometry-extraction
    relation: depends-on
  - slug: revit-symbol-vs-instance-geometry
    relation: composes-with
---

# 不是所有族实例都包含 GeometryInstance（局部连接/相交时 Solid 浮到顶层）

## R — 原文 (Reading)

> 并非所有 Family 实例都包含 GeometryInstance。当 Revit 需要为给定实例生成族几何图形的唯一副本（由于局部连接、相交和实例放置相关的其他因素影响）时，将不包含 GeometryInstance；代之以 Solid 出现在层次结构的顶层。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.2 节（约 p218）

---

## I — 方法论骨架 (Interpretation)

多数文档示例会告诉你：族实例的几何树顶层是一个 GeometryInstance，再往下才是实例化的体。但这个"默认结构"有一个会咬人的例外：当 Revit 因为局部连接、构件相交、实例放置等原因，必须为这个实例生成"唯一副本"几何时，GeometryInstance 会从树里消失，Solid 直接出现在 GeometryElement 的顶层。

换句话说，同一段遍历代码，在独立的梁上能跑通，换一根与墙连接的梁，几何树的结构就变了。这是个反直觉的运行时行为，靠"默认结构"假设写的提取代码会静默丢几何。健壮的写法不预设结构，而是对顶层每个 GeometryObject 做类型分派：是 GeometryInstance 就递归展开，是 Solid/Curve/Mesh 就直接处理——两条路都要写。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 梁几何的两种形态（3.7.5）
- **问题**: 遍历梁族实例几何时结果不稳定。
- **方法论的使用**: 注意到 GeometryElement 可能包含 Solid 或 GeometryInstance，取决于该梁是与其他图元连接还是独立。
- **结论**: 同一机制在"一般族实例"与"结构梁"两类对象上独立复现，是通用规则而非个别特例。
- **结果**: 印证性表述"取决于连接与否"为健壮提取提供依据。

### 案例 2: 钢筋长度计算（3.7.2 代码 3-56）
- **问题**: 柱内部求交分析要拿到体。
- **方法论的使用**: 遍历时对顶层对象做类型分支，Solid 直接取、GeometryInstance 则展开。
- **结论**: 类型分派是唯一可靠的提取方式。
- **结果**: 见 s3-c01。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写通用几何提取工具，担心某些实例的几何结构不同。
2. 遍历连接/相交处的构件（梁与柱、墙与墙）时发现"少了几何"。
3. 调试时发现顶层拿到 Solid 而非预期的 GeometryInstance，怀疑代码写错。
4. 从零实现"递归展开几何树"的公共函数。

### 语言信号 (用户的话里出现这些就应激活)

- "为什么拿到的是 Solid 而不是 GeometryInstance" / "why do I get a Solid instead of a GeometryInstance"
- "遍历梁/柱的几何结果不完整" / "geometry traversal misses solids for joined elements"
- "GeometryInstance 什么时候不存在" / "when is GeometryInstance absent"
- "写一个通用的几何提取函数" / "write a generic geometry extraction helper"

### 与相邻 skill 的区分

- 与 `revit-element-geometry-extraction`：依赖其正向几何提取流程，本 skill 专讲其中"顶层类型不唯一"的反例及修正。
- 与 `revit-symbol-vs-instance-geometry`：本 skill 讲 GeometryInstance 可能根本不存在，与后者的副本语义判别组合诊断。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **改写遍历为类型分派**
   - 完成标准: 对 GeometryElement 顶层每个 GeometryObject 用 is/as 分派：GeometryInstance 走递归展开，Solid/Curve/Mesh 直接收集/处理，没有 else 空分支。

2. **递归展开 GeometryInstance**
   - 完成标准: 对 GeometryInstance 调用 GetInstanceGeometry(instance.Transform)，返回的 GeometryElement 重新进入步骤 1，直到无新实例；同时保留顶层直接出现的 Solid。

3. **自检边界场景**
   - 完成标准: 用一根"与墙连接的梁"与一根"独立梁"各跑一遍，两者提取结果均完整；若只实现了 GeometryInstance 分支，补上 Solid 分支。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 处理对象不是族实例（如系统族墙板）：它们本来就没有 GeometryInstance 概念。
- 明确只需要嵌套族的变换信息：那时只关注 GeometryInstance 即可，但需先确认无顶层 Solid。

### 作者在书中警告的失败模式

- "族实例几何顶层一定是 GeometryInstance"是最普遍的错误预期——连接/相交触发唯一副本后 Solid 顶替出现。
- 只处理 GeometryInstance 的遍历在连接场景下静默丢几何，且不报错，极难排查。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：该行为是引擎内部决策（何时生成唯一副本）的外显，触发条件（连接、相交、实例放置相关因素）在新版本中仍适用，但具体边界可能随版本微调，应以实测为准。

### 容易混淆的邻近方法论

- GeometryInstance 与 Solid 并存 vs 互斥：某些场景顶层可同时出现两类对象，不是"二选一"。
- "唯一副本"与 GetInstanceGeometry 的"副本"：前者是引擎生成的唯一几何，后者是每次都返回的新副本，两个"副本"含义不同。

---

## 相关 skills

- **revit-element-geometry-extraction**（depends-on）：依赖其正向几何提取流程，本 skill 专讲其中"顶层类型不唯一"的反例及修正。
- **revit-symbol-vs-instance-geometry**（composes-with）：本 skill 讲 GeometryInstance 可能根本不存在，与后者的副本语义判别组合诊断。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
