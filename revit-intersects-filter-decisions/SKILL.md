---
name: revit-intersects-filter-decisions
description: |
  几何级精确相交判断（碰撞/干扰检查）用 ElementIntersectsElementFilter（图元间）或 ElementIntersectsSolidFilter（图元 vs Solid）。两者皆 SlowFilter，须先加 QuickFilter；不可反转，取"不相交"用 ExclusionFilter 间接实现。信号："碰撞/干扰检查 / clash detection"、"与实体相交 / intersects solid"、"取不相交 / not intersecting"。
  Trigger: ElementIntersectsElementFilter / ElementIntersectsSolidFilter。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.1.6（约p072-073）
tags: [revit-api, intersection, slow-filter, clash-detection, geometry]
related_skills:
  - slug: revit-filter-quick-slow-logical
    relation: depends-on
---

# 图元相交过滤器决策：ElementIntersectsElement / ElementIntersectsSolid（慢速）

## R — 原文 (Reading)

> ElementIntersectsElementFilter、ElementIntersectsSolidFilter 两个慢速过滤器。
> 这两个过滤器均不会反转匹配在目标对象体之外的图元。
>
> — 宦国胜, 第2章 2.1.6 / 代码 2-16（约p072-073）

---

## I — 方法论骨架 (Interpretation)

要做真正的几何相交判断（不是 bbox 近似），Revit 提供两个过滤器：

- **ElementIntersectsElementFilter**：判断目标图元与另一个图元的实体是否相交。适合"与这根梁相撞的有哪些"。
- **ElementIntersectsSolidFilter**：判断图元与一个任意构造的 Solid 是否相交。适合"与这块预留洞口/自定义体量相撞的有哪些"。判断逻辑与 Revit 自带的干扰检查（Interference Check）一致。

三条硬约束决定使用方式：

1. **都是 SlowFilter**：每个候选图元都要展开实体做几何运算，必须与 QuickFilter 组合（如先 `WhereElementIsNotElementType()`、类别过滤器缩圈），不能全文档裸跑。
2. **都不可反转**：不能直接表达"不相交"——书中明说"不会反转匹配在目标对象体之外的图元"。要补集就用 ExclusionFilter 把相交集合从全集里减掉。
3. **成本随候选集线性增长**：候选集越小越好，bbox 快筛先行是常见前置手段。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 干扰检查实例
- **问题**: 第2章干扰检查示例要找出与指定图元发生几何干扰的图元。
- **方法论的使用**: 用 ElementIntersectsSolidFilter 做几何级判断，结果以 ToElementIds() 返回。
- **结论**: API 干扰检查与 Revit 界面干扰检查共用同一判断逻辑。
- **结果**: 得到与软件内置功能一致的干扰结果 ID 列表。

### 案例 2: 自定义 Solid 的碰撞检查
- **问题**: 需要检查一个程序构造的 Solid（非图元）与所有图元的碰撞。
- **方法论的使用**: `ElementIntersectsSolidFilter(customSolid)` + `WhereElementIsNotElementType()`（QuickFilter）组合，绝不单独使用。
- **结论**: "自定义体量 + QuickFilter 缩圈"是标准结构。
- **结果**: 碰撞检查在大模型上仍可运行。

### 案例 3: bbox 误报的精确替代
- **问题**: BoundingBox 过滤对斜墙等异形元素误报。
- **方法论的使用**: 把 ElementIntersectsSolidFilter 作为几何级复核手段，只作用于 bbox 筛出的小候选集。
- **结论**: 精确判断交给 SlowFilter，成本控制交给前置缩圈。
- **结果**: 误报消除，整体性能可控。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写碰撞/干扰检查代码，要判断图元之间或图元与自定义体量的真实几何相交。
2. bbox 空间过滤出现误报，需要升级到几何级精确判断。
3. 想取"与某对象不相交的所有图元"（反向集合），撞上不可反转约束。
4. 想让 API 检查结果与 Revit 界面的干扰检查功能对得上号。

### 语言信号 (用户的话里出现这些就应激活)

- "碰撞检查 / 干扰检查 / clash detection / interference check"
- "和这个实体相交 / intersects solid / element intersects element"
- "取不相交的集合 / elements NOT intersecting / invert filter"
- "bbox 误报后怎么精确判断 / precise intersection"

### 与相邻 skill 的区分

- 与 `revit-boundingbox-filter-tradeoffs` 的区别: bbox 是低成本近似初筛；本 skill 是高成本精确判断——典型关系是"bbox 先行、相交复核"。
- 与 `revit-filter-quick-slow-logical` 的区别: 三类策略给了"SlowFilter 必须与 QuickFilter 组合"的通用规则；本 skill 是该规则在几何相交场景的落地与"不可反转"特例。
- 与 `revit-filtered-collector-three-steps` 的区别: 本 skill 不讨论构造范围，只讨论相交过滤器本身的行为约束。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定相交目标类型**：目标是另一个图元 → ElementIntersectsElementFilter；目标是程序构造的 Solid → ElementIntersectsSolidFilter。
   - 完成标准: 过滤器类选定，目标对象就绪。

2. **组合 QuickFilter 缩圈**：在相交过滤器之前至少加一个 QuickFilter（类别过滤器、WhereElementIsNotElementType，或 bbox 快筛出的候选集经 IdSet 构造）。
   - 完成标准: 过滤链中没有"单独 SlowFilter"结构。
   - 判停条件: 若候选集已极小（几十个已知 ID），可直接执行相交过滤。

3. **处理反向需求**：若用户要"不相交"集合——先取相交集合的 ID，再用 ExclusionFilter（或全集减相交集）间接构造，禁止对相交过滤器直接施加 Not。
   - 完成标准: 代码中不存在对不可反转过滤器的反转调用；反向需求经补集路径实现。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只要空间粗筛、允许近似——用 BoundingBox 过滤器，别为不需要的精确度买单。
- 判断的是"点是否在图元上"（如拾取命中）——那是 Reference/拾取 API 的职责，不是实体相交过滤。

### 作者在书中警告的失败模式

- SlowFilter 单独全文档运行：几何展开成本叠加，性能崩塌。
- 试图反转相交过滤器取补集：API 明确不支持，静默失败或抛错。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：几何相交判断精度与性能在新版有改进（更快的几何内核路径），但"慢速、需组合、不可反转"三条约束的描述仍基本成立。

### 容易混淆的邻近方法论

- ElementIntersectsElementFilter 比较的是**实体几何**，BoundingBoxIntersectsFilter 比较的是**包围框**——名字都带 Intersects，语义差一个精度层级，混用是常见事故源。

---

## 相关 skills

- revit-filter-quick-slow-logical：depends-on——相交过滤器是 SlowFilter，必须按三类策略先配 QuickFilter 缩圈，本 skill 是该规则在几何相交场景的落地与"不可反转"特例。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
