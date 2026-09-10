---
name: revit-filter-quick-slow-logical
description: |
  给 FilteredElementCollector 挑过滤器按三类组合：QuickFilter（低内存）永远先行缩小集合；SlowFilter（参数/相交等）不能单独用，须叠加在 QuickFilter 后；LogicalFilter（And/Or/Not）做逻辑组合。Revit 自动重排；部分过滤器不可反转。信号："按参数值过滤 / parameter filter"、"性能慢 / collector too slow"。
  Trigger: ElementQuickFilter / ElementSlowFilter / parameter filter。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.1.2（约p062-068）
tags: [revit-api, element-filter, performance, quick-filter, slow-filter]
related_skills:
  - slug: revit-filtered-collector-three-steps
    relation: depends-on
---

# ElementFilter 三类策略选择：QuickFilter / SlowFilter / LogicalFilter

## R — 原文 (Reading)

> ElementQuickFilter：快速过滤器仅对 ElementRecord 进行操作，是一个低内存占用的类。
> ElementSlowFilter：慢速过滤器首先需要获取图元并展开到内存中。
> ElementLogicalFilter：逻辑过滤器逻辑组合两个或更多过滤器。
>
> — 宦国胜, 第2章 2.1.2 / 表 2-1、2-3、2-4（约p062-068）

---

## I — 方法论骨架 (Interpretation)

Revit 的所有 ElementFilter 分三类，性能特征完全不同：

- **QuickFilter（快速）**：只读数据库层的 ElementRecord，不把图元展开到内存，代价极低。类别过滤器、类过滤器、WhereElementIsNotElementType 等都属于这一类。
- **SlowFilter（慢速）**：必须把图元完整展开到内存才能判断，代价高。参数过滤器（ElementParameterFilter）、几何相交过滤器都在此类。
- **LogicalFilter（逻辑）**：把两个以上过滤器用 AND/OR/NOT 组合。

由此推出组合规则：**SlowFilter 永远不能单独使用**。正确姿势是先用一个或多个 QuickFilter 把候选集缩小（比如先 `ElementClassFilter(typeof(Room))` + `WhereElementIsNotElementType()`），再叠加 SlowFilter 做精细判断。好消息是 Revit 会自动重排过滤器的执行顺序，快慢次序写错也能被优化，但"有没有至少一个 QuickFilter"这个前提不能省。

另一个硬约束：部分过滤器**不可反转**（如 RoomFilter、FamilyInstanceFilter、相交过滤器），不能用 Not 取"非"逻辑——想要补集要用 ExclusionFilter 间接实现。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 按参数值过滤房间
- **问题**: 找出所有面积大于 100 的房间。
- **方法论的使用**: ElementParameterFilter 是 SlowFilter，先加 `ElementClassFilter(typeof(Room))` + `WhereElementIsNotElementType()` 缩小集合，再叠参数过滤器。
- **结论**: "Quick 缩圈 + Slow 精筛"是参数过滤的标准结构。
- **结果**: 过滤在大集合上仍保持可接受性能。

### 案例 2: 干扰检查中的相交过滤
- **问题**: 第2章 2.1.6 干扰检查实例需要几何级相交判断。
- **方法论的使用**: ElementIntersectsSolidFilter 属 SlowFilter，按规则与 QuickFilter 组合后使用。
- **结论**: 几何精确判断的代价由组合结构控制。
- **结果**: 干扰检查代码既精确又不至于全文档展开。

### 案例 3: 不可反转过滤器的限制
- **问题**: 想用"非"逻辑排除某些族实例/族符号。
- **方法论的使用**: 书中标注 FamilyInstanceFilter/FamilySymbolFilter 不可反转，改用其他集合运算间接实现。
- **结论**: 写 Not 前先确认该过滤器支持反转。
- **结果**: 避免运行时因不支持反转而失败。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要按参数值（面积、名称、自定义参数）过滤图元，正准备直接上 ElementParameterFilter。
2. collector 跑得很慢，怀疑是过滤器组合不当（典型的"只有 SlowFilter"结构）。
3. 想对某个过滤器取反（"所有不是房间的图元"），需要先确认它能否反转。
4. 在大型模型（几万到百万图元）上写检索，需要预先设计过滤层级控制成本。

### 语言信号 (用户的话里出现这些就应激活)

- "按参数过滤 / parameter filter / filter by parameter value"
- "过滤很慢 / collector slow / performance issue with filter"
- "取反 / not filter / invert filter / exclude category"
- "QuickFilter SlowFilter 区别 / quick vs slow filter"

### 与相邻 skill 的区分

- 与 `revit-filtered-collector-three-steps` 的区别: 三步构建管"collector 怎么搭、范围怎么定"；本 skill 管第二步"过滤器怎么选、怎么组合"。
- 与 `revit-boundingbox-filter-tradeoffs` 的区别: BoundingBox 过滤器是 QuickFilter 里的一个具体家族，本 skill 是它们背后的选择框架。
- 与 `revit-intersects-filter-decisions` 的区别: 相交过滤器是 SlowFilter 的具体家族，本 skill 决定它"必须和谁组合"。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **给每个目标过滤器分类**：查它属于 Quick / Slow / Logical 哪类（书中表 2-1/2-3/2-4 是对照表）。
   - 完成标准: 每个过滤器旁标注类别。

2. **检查组合结构**：确认过滤器链中至少有一个 QuickFilter 且排在效果上能大幅缩圈（类别/类型级），SlowFilter 只做最后精筛。
   - 完成标准: 不存在"单独 SlowFilter"或"全 SlowFilter 链"结构。
   - 判停条件: 若目标集合本来就极小（如 IdSet 构造的几十个候选），SlowFilter 单独使用可接受，跳到步骤 3。

3. **校验反转合法性**：对每个准备用 Not/反转的过滤器，确认其支持反转；不支持则改用 ExclusionFilter 或补集思路。
   - 完成标准: 代码中不存在对不可反转过滤器施加反转逻辑的调用。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需要一个类别过滤器的具体用法——直接看 API 即可，不必做全链路策略设计。
- 讨论的是某个具体过滤器家族的行为（bbox 误报、相交不可反转）——转对应专属 skill。

### 作者在书中警告的失败模式

- SlowFilter 单独作用于全文档：所有图元被展开到内存，性能急剧劣化。
- 对不可反转的过滤器使用"非"逻辑：直接失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版补充了更多 QuickFilter（部分原 Slow 判定前移），且提供了 FilteredElementCollector 的 Where/LINQ 链式写法，但快慢分离与自动重排的核心机制未变。

### 容易混淆的邻近方法论

- "结果获取方式"（ToElements/FirstElement）影响的是过滤后的物化成本，与本 skill 的过滤执行成本是两个独立的性能维度，不要混为一谈。

---

## 相关 skills

- revit-filtered-collector-three-steps：depends-on——本 skill 是三步构建流程中第二步"叠加过滤器"的内部决策，前提是 collector 骨架已就绪。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
