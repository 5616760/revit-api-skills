---
name: revit-element-retrieval-four-entries
description: |
  需要"拿到图元"时先用四入口决策框架：已知单个对象用 ElementId/GetElement（最快）；批量同类用 FilteredElementCollector；用户交互选择用 Selection/PickObject；项目级唯一对象（标高/视图/材质）用 Document 属性直接访问。信号："怎么获取图元 / how to get elements"、"已知 ID 回查"、"遍历所有墙 / iterate elements"。不适用：具体过滤或选择细节（下游 skill）。
  Trigger: get element / retrieve elements / ElementId lookup。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.5.3（约p072-073）
tags: [revit-api, element-retrieval, decision-framework, filteredelementcollector, selection]
related_skills:
  - slug: revit-elementid-vs-uniqueid
    relation: composes-with
---

# 图元检索四法决策框架

## R — 原文 (Reading)

> 用 Revit API 检索图元有以下几种方法：ElementId（图元 ID）；Element Filtering and Iteration（图元过滤和迭代）；Selected Elements（选定图元）；Specific Elements（特定图元，某些图元可作为文件的属性）。
>
> — 宦国胜, 第1章 1.5.3（约p072-073）

---

## I — 方法论骨架 (Interpretation)

在写任何"找图元"的 Revit 代码前，先回答三个问题：这个图元是已知还是未知？是单个还是批量？由程序自动定位还是用户交互指定？

三个问题的答案对应四条检索路径：

1. **已知具体对象** → 用 ElementId（或 UniqueId）直接调 `Document.GetElement()`，O(1) 命中，最快。
2. **只知道特征（类别、类型、参数）** → 用 FilteredElementCollector 做过滤迭代，适合批量。
3. **需要用户在界面里点选** → 用 Selection API（Selection.Elements 或 PickObject 系列）。
4. **项目级唯一对象**（标高、视图、材质、族类型集合）→ 它们挂在 Document 的属性上（如 `doc.Settings.Materials`、`doc.Levels`），不需要过滤。

选错入口的代价很实际：用过滤去取一个已知 ID 的对象是浪费；用 GetElement 去找"所有面积大于 100 的房间"根本做不到。若要把图元 ID 存到外部数据库再回查，应存 UniqueId（GUID）而非 ElementId，因为 ElementId 只在当前项目内稳定。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 已知 ID 回查外部存储
- **问题**: 在外部数据库中存了图元 ID，之后要回查该图元。
- **方法论的使用**: 按"已知单对象"分支选 ElementId 路径；但跨会话/跨文件时 ElementId 不稳定，升级决策为存 UniqueId。
- **结论**: 外部存储用 UniqueId（GUID），回查时经 Document 检索。
- **结果**: 跨文件/跨会话都能可靠回查，不因项目变化失效。

### 案例 2: 批量同类图元检索演练
- **问题**: 第1章 1.2.5 的过滤检索演练需要找出某类图元。
- **方法论的使用**: 按"未知、批量"分支选 Element Filtering，用 `FilteredElementCollector(document)` 构造过滤。
- **结论**: 批量同类检索走过滤路径，不走 GetElement。
- **结果**: 一次调用拿到全部目标图元集合。

### 案例 3: 用户选集获取
- **问题**: 第2章 2.2 的示例需要拿到用户在界面上已选的对象。
- **方法论的使用**: 按"用户交互"分支选 Selection 路径，读 `UIDocument.Selection.Elements`。
- **结果**: 程序正确获得用户当前选集，无需任何过滤。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开始写一段新代码，第一句就是"我要先拿到这些图元"，但还没想清楚怎么拿。
2. 把 Revit 图元 ID 导出到外部数据库（BIM 数据管道、模型对比），纠结存哪种 ID。
3. 发现自己用 FilteredElementCollector 全文档扫了一遍只为找一个已知 ID 的图元（或反过来用 GetElement 猜 ID）。
4. 要取项目里唯一的对象（当前视图、某个材质、标高集合），却在一堆过滤条件里打转。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么获取/检索图元" / "how to get/retrieve elements"
- "已知 ID 怎么反查" / "lookup element by id"
- "遍历所有 XX" / "iterate all walls / find all elements"
- "把图元 ID 存到数据库" / "export element id to database"

### 与相邻 skill 的区分

- 与 `revit-filtered-collector-three-steps` 的区别: 本 skill 只做"选哪条路"的顶层决策；一旦决定走过滤路径，构造细节交给 collector 三步构建 skill。
- 与 `revit-elementid-vs-uniqueid` 的区别: 本 skill 只在决策点提示"外部存储选 UniqueId"；两套 ID 的完整对比与陷阱是专属 skill 的职责。
- 与 `revit-selection-pickobject-filter` 的区别: 本 skill 决定"要不要走用户交互"；拾取 API 的具体用法属选择集 skill。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定检索需求类型**：问清三个问题——已知还是未知？单个还是批量？自动还是用户交互？
   - 完成标准: 明确写下需求属于 {已知单对象, 批量同类, 用户交互, 项目级唯一} 之一。

2. **按分支选入口**：
   - 已知单对象 → `Document.GetElement(ElementId)`；跨会话存储改用 UniqueId。
   - 批量同类 → `FilteredElementCollector`（转到三步构建 skill）。
   - 用户交互 → `UIDocument.Selection` / `PickObject` 系列（转到选择集 skill）。
   - 项目级唯一 → `Document` 对应属性（Settings、Levels、Views 等）。
   - 完成标准: 代码中检索入口与需求类型一一对应，无"用过滤找已知 ID"这类错配。
   - 判停条件: 若需求同时含多种（如先过滤再让用户挑），分别走各分支后组合，不要强行合并成一个入口。

3. **验证入口正确性**：检查是否有更便宜的路径（GetElement 能解决的不要过滤；Document 属性能解决的不要遍历）。
   - 完成标准: 所选入口是四条路径中代价最低的一条，并能说明为什么。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已经确定走过滤路径，讨论的是过滤器怎么组合——那是 ElementFilter 三类策略 skill 的事。
- 讨论的是拾取交互的细节（如何限制用户只能选平面）——那是选择集 skill 的事。
- 纯粹问"ElementId 和 UniqueId 有什么区别"的知识性问题——直接转双 ID 对比 skill。

### 作者在书中警告的失败模式

- 把 ElementId 当跨项目唯一标识存外部系统：ElementId 仅项目内唯一，跨文件/会话不稳定，回查会失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 API：四入口框架至今成立，但新版本补充了更多 Document 快捷属性与 FilteredElementCollector 扩展（如LINQ集成更成熟），实际可选项略多于书中列举。

### 容易混淆的邻近方法论

- "图元检索四法"是入口选择；"FilteredElementCollector 三步构建"是其中一条路径的展开——不要在决策层就直接跳进过滤器细节。

---

## 相关 skills

- revit-elementid-vs-uniqueid：composes-with——四法检索拿到 ElementId 后，双 ID 选型决定结果如何落地（进程内传参用 ID、外部持久化用 UID），两者组合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
