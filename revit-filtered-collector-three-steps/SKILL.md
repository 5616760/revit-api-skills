---
name: revit-filtered-collector-three-steps
description: |
  构造 FilteredElementCollector 走"三步构建"：新建 collector（Document 全文档 / IdSet 候选 / View 仅视图可见）→ 施加过滤器 → 获取图元或 ID。视图构造前先调 IsViewValidForElementIteration。信号："怎么用过滤器找图元 / filtered element collector"、"只在当前视图找 / elements visible in view"。不适用：单个已知对象的 GetElement、快慢过滤器细节。
  Trigger: FilteredElementCollector / view-based collector / 三步构建。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.1（约p061-062）
tags: [revit-api, filteredelementcollector, element-filter, query, decision-framework]
related_skills:
  - slug: revit-element-retrieval-four-entries
    relation: depends-on
---

# 图元过滤决策树：FilteredElementCollector 三步构建

## R — 原文 (Reading)

> 通过指定过滤器获取图元的基本步骤如下：
> （1）新建一个 FilteredElementCollector。
> （2）对它运用一个或多个过滤器。
> （3）获取过滤的图元或图元 ID。
>
> — 宦国胜, 第2章 2.1 / 代码 2-1（约p061-062）

---

## I — 方法论骨架 (Interpretation)

FilteredElementCollector 是 Revit 里"按条件找图元"的标准工具，但它不是拿来就用的——第一个决策点是构造函数，因为构造函数直接决定检索范围：

1. `new FilteredElementCollector(document)`：全文档，含不可见与非图形图元。
2. `new FilteredElementCollector(document, idSet)`：只在已知的一批 ElementId 里过滤，适合"已有候选名单再筛选"。
3. `new FilteredElementCollector(document, view.Id)`：只遍历该视图可见的图元——用前必须调 `FilteredElementCollector.IsViewValidForElementIteration(view)` 校验视图类型，否则访问会异常。

范围定了之后才进入三步流程：新建 collector → 叠加过滤器（一个或多个）→ 用 ToElements/ToElementIds 等方法取结果。

把"构造函数选择"当成检索范围选择来对待，就能避开最常见的错误：在三维视图级 collector 里找标高（找不到，因为不可见），或在文档级 collector 里找"用户看得见的墙"（找到一堆不可见的）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 第1章过滤检索演练
- **问题**: 演练程序要按条件找出目标图元集合。
- **方法论的使用**: 按三步走——`FilteredElementCollector(document)` 构造（文档级范围）→ 施加过滤器 → 取结果。
- **结论**: 文档级构造 + 过滤器叠加是默认检索模式。
- **结果**: 演练代码简洁地拿到全部符合条件的图元。

### 案例 2: 检索所有标高
- **问题**: 第3章 3.5.1 示例要统计全文档的标高。
- **方法论的使用**: 标高是基准图元，多数视图不可见，所以选 Document 构造（文件级），再 `collector.OfClass(typeof(Level)).ToElements()`。
- **结论**: 目标图元可能不可见时，禁止用视图级构造。
- **结果**: 正确拿到全部标高，不受当前视图可见性影响。

### 案例 3: 视图级检索的校验
- **问题**: 需要只检索当前视图中可见的图元。
- **方法论的使用**: 用 `new FilteredElementCollector(document, view.Id)` 构造，且构造前先调 `IsViewValidForElementIteration(view)` 检查。
- **结论**: 视图级构造 = 可见性过滤的前提是视图本身支持元素迭代。
- **结果**: 避免在无效视图上访问导致异常。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 第一次写"找出所有墙/门/房间"类代码，需要搭检索骨架。
2. 需求明确是"当前视图里可见的图元"（如只处理当前平面视图的标注），要选视图级构造并校验。
3. 手里已有一批 ElementId（如上次过滤的结果或外部输入），要在其中二次筛选——用 IdSet 构造而不是重新全文档扫。
4. 出现"为什么我的 collector 找不到标高/找不到隐藏图元"这类 bug，需要回头检查构造函数选错了范围。

### 语言信号 (用户的话里出现这些就应激活)

- "用过滤器获取图元的步骤 / FilteredElementCollector 怎么用"
- "只在当前视图里找 / only visible in view / view-based collector"
- "从这些 ID 里再筛 / filter from a set of ids"
- "为什么收集器找不到 / collector returns nothing"

### 与相邻 skill 的区分

- 与 `revit-element-retrieval-four-entries` 的区别: 那个决定"走不走过滤这条路"；本 skill 在已决定过滤后解决"构造函数选哪个、骨架怎么搭"。
- 与 `revit-filter-quick-slow-logical` 的区别: 本 skill 只管三步骨架与范围；过滤器选 Quick 还是 Slow、怎么组合是下一个决策层。
- 与 `revit-view-vs-document-collector` 的区别: 本 skill 提到三种构造；视图级 vs 文件级的行为差异（几何重建代价、拿不到非图形图元）由该 skill 深入展开。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **选构造函数**：判断检索范围——全文档用 Document 构造；已知候选 Id 用 IdSet 构造；仅当前视图可见用 View 构造。
   - 完成标准: 明确写出所选构造函数及理由。
   - 判停条件: 若目标图元可能是不可见图元（标高、线荷载等），禁止选 View 构造，强制回到 Document 构造。

2. **校验视图（仅 View 构造）**：调用 `FilteredElementCollector.IsViewValidForElementIteration(view)`，返回 false 则改用 Document 构造。
   - 完成标准: View 构造路径必有校验语句；校验失败有降级方案。

3. **叠加过滤器并取结果**：依次施加类别/类型/参数过滤器，最后调 ToElements() 或 ToElementIds() 取结果。
   - 完成标准: 结果只取一次并存入变量（collector 不缓存，重复调用会重复过滤）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需拿一个已知 ID 的对象——直接 `Document.GetElement()`，不必构造 collector。
- 讨论过滤器性能组合（Quick 先行、Slow 后置）——转 ElementFilter 三类策略 skill。
- 需要用户点选而非程序过滤——转选择集 skill。

### 作者在书中警告的失败模式

- 视图级构造不校验 `IsViewValidForElementIteration` 就直接访问，会抛异常。
- 视图级 collector 拿不到非图形图元和不可见图元——在视图中找标高注定失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 API 中 collector 的LINQ扩展与部分过滤器有增补，但三种构造函数决定范围的核心设计未变。

### 容易混淆的邻近方法论

- "三步构建"是骨架；"结果获取方式"（ToElements vs FirstElement vs 迭代器）是第三步内部的独立决策，有专属 skill。

---

## 相关 skills

- revit-element-retrieval-four-entries：depends-on——本 skill 是"图元检索四法"中过滤路径的内部展开，前提是已决定走 FilteredElementCollector。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
