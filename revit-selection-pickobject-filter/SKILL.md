---
name: revit-selection-pickobject-filter
description: |
  用户交互式选图元用选择集 API：当前选集 Selection.Elements；拾取 PickObject/PickObjects、PickElementsByRectangle；限制可选范围用 ISelectionFilter（如只选 PlanarFace）。信号："让用户选择 / let user pick"、"只能选平面 / only allow planar faces"。不适用：程序自动过滤（FilteredElementCollector）。
  Trigger: PickObject / ISelectionFilter / Selection.Elements。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.2（约p073-078）
tags: [revit-api, selection, uidocument, picking, iselectionfilter]
related_skills:
  - slug: revit-element-retrieval-four-entries
    relation: depends-on
  - slug: revit-filtered-collector-three-steps
    relation: contrasts-with
  - slug: revit-document-vs-uidocument
    relation: depends-on
---

# 选择集操作决策：Selection.Elements / PickObject / PickBox / ISelectionFilter

## R — 原文 (Reading)

> 使用 UIDocument.Selection.Elements 属性，可从当前活动文件获取所选对象。
> PickObject() 提示用户选择一个 Revit 模型中的对象。PickObjects() 提示用户选择多个 Revit 模型中的对象。
>
> — 宦国胜, 第2章 2.2 选集 / 代码 2-17~2-22（约p073-078）

---

## I — 方法论骨架 (Interpretation)

凡是"让用户决定操作对象"的代码都走选择集 API，分三种模式：

1. **查询模式**：用户已在界面里选好了东西，程序读 `UIDocument.Selection.Elements` 直接拿到当前选集。最被动，也最不打断用户。
2. **拾取模式**：程序主动提示用户选——`PickObject/PickObjects`（点选，单/多）、`PickElementsByRectangle`（矩形框选）、`PickBox`（框出区域返回 PickedBox，可转 Outline 做空间过滤）、`PickPoint`（只取点）。程序暂停等待用户动作，适合交互式命令。
3. **过滤模式**：拾取时限制用户能选什么——实现 `ISelectionFilter` 接口，它有两个粒度：`AllowElement`（这个图元整体可不可选）与 `AllowReference`（这个引用/子对象可不可选，如某个面）。把它传给 PickObject 系列，不符合条件的对象根本点不中。

典型组合拳：PickObjects(ObjectType.Face, 提示语, selectionFilter) —— 让用户只能选到特定类型的面。注意拾取类 API 必须在 UI 上下文（有 UIDocument）中使用，这是它与 collector 类过滤的本质区别。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 获取用户当前选集
- **问题**: 第1章 1.2.4 示例需要拿到用户已经选好的图元做处理。
- **方法论的使用**: 查询模式——读 `Selection.Elements`，不做任何过滤。
- **结论**: 已有选集直接读，不重复要求用户再选。
- **结果**: 程序正确处理用户预先选定的对象。

### 案例 2: 强制只选平面
- **问题**: 命令要求用户只能拾取平面（PlanarFace），曲面和其他面一律不可选。
- **方法论的使用**: 实现 ISelectionFilter，在 `AllowReference(ref, xyz)` 里用 `GetGeometryObjectFromReference(ref) as PlanarFace` 判断，非平面返回 false；调用 `PickObjects(ObjectType.Face, prompt, filter)`。
- **结论**: 过滤逻辑放在选择器里，不放在拾取后的结果校验里。
- **结果**: 用户界面上非平面对象直接选不中，无需事后报错。

### 案例 3: 框选区域接入空间过滤
- **问题**: 用户要框选一块区域，程序找出区域内图元。
- **方法论的使用**: PickBox 返回 PickedBox，构造 Outline 交给 BoundingBox 过滤器。
- **结论**: 交互式框选是空间过滤的天然输入源。
- **结果**: "用户框选 → bbox 过滤"流水线打通。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 命令的语义是"对用户选的对象做 XX"，需要拿当前选集或提示用户选。
2. 要限制用户可选范围（只选墙、只选平面、排除链接文件），避免选错后报错。
3. 需要"点一下取一个面/一条边"这类子对象级选择（ObjectType.Face/Edge/Point）。
4. 用户框选屏幕区域后要对区域内图元做批量处理。

### 语言信号 (用户的话里出现这些就应激活)

- "让用户选 / 提示用户选择 / prompt user to pick / pick object"
- "只能选平面/墙 / only allow planar faces / restrict selection"
- "获取选中的图元 / get currently selected elements"
- "框选 / pick by rectangle / PickBox"

### 与相邻 skill 的区分

- 与 `revit-element-retrieval-four-entries` 的区别: 四法框架决定"要不要走交互路径"；本 skill 展开交互路径内部的模式与过滤器选择。
- 与 `revit-boundingbox-filter-tradeoffs` 的区别: PickBox 负责产出框；对框做过滤是 bbox skill 的职责。
- 与 `revit-document-vs-uidocument` 的区别: 拾取 API 必须走 UIDocument 是 UI/DB 分工的具体体现；分工总原则见该 skill。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定选择模式**：用户已选好（查询 Selection.Elements）/ 需要提示选择（PickObject 系列）/ 需要区域（PickBox）。
   - 完成标准: 模式确定，明确要不要打断用户。

2. **决定是否需要 ISelectionFilter**：若要限定可选类型/子对象，实现 AllowElement/AllowReference；子对象级限定（如只要平面）用 AllowReference 做几何类型判断。
   - 完成标准: 过滤器类的判定逻辑可单独说明；无限制则显式不传。
   - 判停条件: 若用户可自由选择任意对象，跳过本步。

3. **调用拾取并转换结果**：PickObject/PickObjects 返回 Reference，用 `doc.GetElement(reference)` 取图元；PickBox 结果转 Outline 交给空间过滤。
   - 完成标准: 每个返回的 Reference 都被正确解析为 Element（及所需子对象）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 对象由程序按条件自动确定（所有墙、所有面积>100 的房间）——走 FilteredElementCollector，不要强迫用户手选。
- 在无 UI 上下文的环境（IExternalDBApplication、后台批处理）——PickObject 系列不可用，只能走 DB 层检索。

### 作者在书中警告的失败模式

- 把过滤逻辑放在拾取后的结果校验里让用户反复重选——正确做法是 ISelectionFilter 让不可选对象根本点不中。
- 在 DB 层应用中调用 UI 拾取 API——直接失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 Selection API 有扩展（如 Selection.GetElementIds/选集变更事件、PickObject 的更多重载），但"查询/拾取/过滤"三模式与 ISelectionFilter 双粒度结构未变。

### 容易混淆的邻近方法论

- ISelectionFilter（限制用户能选什么）与 ElementFilter（限定程序过滤什么）是两套体系：前者管交互输入，后者管数据查询，不能互相替代。

---

## 相关 skills

- revit-element-retrieval-four-entries：depends-on——本 skill 是"图元检索四法"中用户交互路径的内部展开。
- revit-filtered-collector-three-steps：contrasts-with——程序自动过滤 vs 用户交互拾取，两套"确定操作对象"机制的边界划分。
- revit-document-vs-uidocument：depends-on——拾取 API 必须经 UIDocument，是 UI/DB 分工律的具体体现。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
