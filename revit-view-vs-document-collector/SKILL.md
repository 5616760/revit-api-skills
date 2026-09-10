---
name: revit-view-vs-document-collector
description: |
  FilteredElementCollector 前先做视图级 vs 文件级决策：视图级（document+view.Id）只含当前视图可见图元，拿不到不可见/非图形图元，首次运行可能触发几何重建导致性能骤降；文件级检索全部图元。基准图元（标高）多数视图不可见，必须文件级。信号："当前视图的图元 / elements in current view"、"三维视图里找不到 / collector finds nothing in 3D view"。不适用：IdSet 二次筛选。
  Trigger: view-based collector / document-level / invisible elements。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.6.1 视图中图元可见性（约p116）
tags: [revit-api, filteredelementcollector, view, visibility, performance]
related_skills:
  - slug: revit-filtered-collector-three-steps
    relation: depends-on
---

# 可见性检索的视图级 vs 文件级 FilteredElementCollector

## R — 原文 (Reading)

> 基于视图的 FilteredElementCollector 只包含当前视图中的可见图元，无法检索到非图形图元或不可见图元。而基于文件的 FilteredElementCollector 检索文件中所有的图元，包括不可见图元和非图形图元。
>
> — 宦国胜, 第2章 2.6.1（约p116）

---

## I — 方法论骨架 (Interpretation)

选 collector 构造函数时，"视图级 vs 文件级"不只是语法差异，而是检索语义的分岔：

- **视图级** `new FilteredElementCollector(document, view.Id)`：结果 = 该视图可见的图形图元。看不见的（被隐藏、当前视图类型不显示的基准图元）全部缺席。副作用：首次在某视图上跑收集器可能触发**视图几何重建**——本来没生成的显示几何要现算，性能骤降。
- **文件级** `new FilteredElementCollector(document)`：结果 = 文档里全部图元，不管可见性、不管是否图形图元。

书中"空项目新建默认三维视图"的例子最能击破直觉：视图里空无一物，文件里却有很多图元——**收集器返回的 ≠ 模型内容**，视图级返回的只是"这个视图画出来的东西"。

实用判断法：目标图元是基准类（标高、轴网）或可能被隐藏——直接文件级；目标语义就是"用户看得见的"（当前平面视图的标注）——视图级，且接受首次几何重建的代价（可预热缓解）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 三维视图找不到标高
- **问题**: 在三维视图里用收集器找标高图元，结果为空，但标高明明存在。
- **方法论的使用**: 依据"视图级只含可见图元"的规则诊断：标高是基准图元，该视图不显示 → 换文件级构造。
- **结论**: 找不到 ≠ 不存在，先查构造函数的可见性语义。
- **结果**: 文件级构造正确返回全部标高。

### 案例 2: 检索所有标高的统计
- **问题**: 第3章 3.5.1 示例要统计全文档标高。
- **方法论的使用**: 标高在多数视图不可见，用文件级构造 + OfClass(typeof(Level))。
- **结论**: 统计类需求（与视图无关）一律文件级。
- **结果**: 统计结果覆盖所有标高，不受当前视图状态影响。

### 案例 3: 首次视图级收集的慢
- **问题**: 首次在视图上跑收集器特别慢，之后又变快。
- **方法论的使用**: 识别为视图几何重建代价——收集器要先把视图显示几何生成出来。
- **结论**: 视图级收集的隐藏成本是几何重建；可用预热（提前跑一次轻量收集）或改文件级规避。
- **结果**: 性能问题被归因，方案二选一。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 收集器在某视图（尤其三维视图）返回空或漏图元，怀疑 API 出 bug。
2. 写"只处理当前视图可见对象"的命令，需要视图级语义并控制性能。
3. 批处理/统计类需求误用了视图级构造，结果随当前激活视图变化、时对时错。
4. 首次运行某视图级命令明显卡顿，需要诊断原因。

### 语言信号 (用户的话里出现这些就应激活)

- "当前视图的图元 / elements in current view / visible elements"
- "收集器找不到图元 / collector returns nothing / missing elements"
- "标高/轴网找不到 / can't find levels"
- "第一次跑很慢 / first run slow / view regeneration"

### 与相邻 skill 的区分

- 与 `revit-filtered-collector-three-steps` 的区别: 三步构建列出三种构造入口；本 skill 专门裁决"View 构造 vs Document 构造"的行为差异与代价，是该决策的深水区。
- 与 `revit-filter-quick-slow-logical` 的区别: 快慢策略管过滤器执行；本 skill 管构造函数的可见性语义与几何重建成本——性能问题的两个不同来源。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定需求是否绑定可见性**：结果必须"用户在当前视图看得见" → 视图级；结果语义是"模型里有什么" → 文件级。
   - 完成标准: 明确写下判定及理由。
   - 判停条件: 若目标图元含基准类（标高/轴网）或可能隐藏——直接文件级，结束。

2. **视图级路径做防护**：先 `IsViewValidForElementIteration(view)` 校验；评估首次几何重建代价，必要时预热或提示用户。
   - 完成标准: 校验语句存在；性能代价有应对（预热/文件级替代）。

3. **验证结果语义**：抽查结果是否与预期语义一致（可见图元全在、隐藏图元不在），尤其确认没有"找不到即不存在"的误判。
   - 完成标准: 空/缺结果时已排除构造函数语义原因，能解释每一类缺席。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已知候选 IdSet 的二次筛选——IdSet 构造与可见性无关。
- 用户交互式选择——Selection 语义由选集 skill 管。

### 作者在书中警告的失败模式

- 把"收集器返回空"当"模型里没有"——视图级构造下这是错误结论（空三维视图反例）。
- 忽视首次视图级收集的几何重建代价，在启动路径上埋下卡顿。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：视图几何生成机制与"打开时不需要生成视图几何"的优化在新版有演进，但"视图级=可见图元+可能触发重建"的语义未变。

### 容易混淆的邻近方法论

- "视图级收集器只含可见图元"与"视图过滤/临时隐藏"是两回事：前者是构造函数语义，后者是视图状态——用户临时隐藏一个图元后，视图级收集器也会漏掉它，排查时要分开考虑。

---

## 相关 skills

- revit-filtered-collector-three-steps：depends-on——三步构建列出三种构造入口，本 skill 深入裁决 View 构造 vs Document 构造的行为差异与性能代价。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
