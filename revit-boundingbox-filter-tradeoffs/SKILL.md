---
name: revit-boundingbox-filter-tradeoffs
description: |
  按空间范围检索图元时，BoundingBox 过滤器（IsInside/Intersects/ContainsPoint + Outline）是低成本 QuickFilter 首选；但它基于边界框而非实际几何，旋转/异形元素 bbox 远大于实际几何会误报，需精确时升级为 ElementIntersectsSolidFilter（SlowFilter）。信号："框选 / bounding box filter"、"结果有误报 / false positives"。
  Trigger: BoundingBoxIntersectsFilter / Outline / false positive。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.1.5（约p071-073）
tags: [revit-api, boundingbox, quick-filter, spatial-query, geometry]
related_skills:
  - slug: revit-filter-quick-slow-logical
    relation: depends-on
  - slug: revit-intersects-filter-decisions
    relation: composes-with
  - slug: revit-selection-pickobject-filter
    relation: composes-with
---

# 边界框过滤器决策：BoundingBox* 三种 + Outline 输入 + 反转陷阱

## R — 原文 (Reading)

> BoundingBoxIsInsideFilter、BoundingBoxIntersectsFilter、BoundingBoxContainsPointFilter。
> BoundingBox 过滤器对图元实际几何形状与它的边界框几何形状紧密匹配过滤效果好。
>
> — 宦国胜, 第2章 2.1.5 / 代码 2-2、2-3（约p071-073）

---

## I — 方法论骨架 (Interpretation)

想按空间位置找图元（"这个区域里有什么""和这个框相交的是什么"），第一候选是 BoundingBox 家族过滤器，共三个：

- **BoundingBoxIsInsideFilter**：图元边界框完全在指定 Outline 内。
- **BoundingBoxIntersectsFilter**：图元边界框与 Outline 相交。
- **BoundingBoxContainsPointFilter**：边界框包含指定点。

输入统一是 Outline（最小最大点围成的框）。它们都是 QuickFilter，成本极低，适合做大范围初筛。

核心权衡在"准"字：过滤器比较的是**边界框**，不是实际几何。当图元几何与它的 bbox 紧贴（轴对齐的直墙、矩形柱）时结果准确；当图元旋转、异形（斜墙、弧形构件）时 bbox 包进大量空白区域，出现**误报**（结果里有实际并不相交的图元）。所以正确用法是两段式：bbox 快筛出候选 → 需要几何级精确时再用 ElementIntersectsSolidFilter 复核。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 与指定区域相交检索
- **问题**: 需要检索与某矩形区域相交的图元。
- **方法论的使用**: 书中代码 2-2/2-3 用 Outline 构造输入，BoundingBoxIntersectsFilter 快速筛选。
- **结论**: 区域检索的默认首选用 bbox 家族（QuickFilter，低内存）。
- **结果**: 大范围候选被低成本筛出。

### 案例 2: 斜墙误报的升级路径
- **问题**: 用 BoundingBoxIntersectsFilter 后发现旋转的斜墙出现在结果里，但实际几何不与区域相交。
- **方法论的使用**: 依据"bbox 过滤 ≠ 几何过滤"的边界认知，对候选集改用 ElementIntersectsSolidFilter 做几何级复核。
- **结论**: bbox 快筛 + 相交过滤器精筛的两段式结构。
- **结果**: 误报被消除，且整体性能仍可控（SlowFilter 只作用于小候选集）。

### 案例 3: 与 PickBox 的组合
- **问题**: 用户在界面框选一块区域，程序要找出其中图元。
- **方法论的使用**: PickBox 返回 PickedBox，由它构造 Outline，喂给 BoundingBox 过滤器。
- **结果**: 用户交互式框选直接接入空间过滤流水线。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需求是"找出某区域/某范围内/与某框相交的图元"，正在选空间过滤工具。
2. 空间过滤结果里混进了明显不在区域内的斜墙、弧形构件——bbox 误报复现。
3. 想先快后准：大模型上先低成本缩圈，再对候选做精确几何判断。
4. 用户用 PickBox 框选了一块屏幕区域，需要转成过滤器输入。

### 语言信号 (用户的话里出现这些就应激活)

- "边界框过滤 / bounding box filter / bbox filter"
- "范围内/相交的图元 / elements inside region / intersecting elements"
- "过滤结果不对/误报 / false positives in filter results"
- "框选 / PickBox / outline filter"

### 与相邻 skill 的区分

- 与 `revit-intersects-filter-decisions` 的区别: bbox 是"快但可能误报"的空间初筛；ElementIntersects* 是"慢但几何精确"的复核——两者常串联，角色不同。
- 与 `revit-filter-quick-slow-logical` 的区别: 本 skill 是 QuickFilter 中空间家族的具体行为；三类策略是它们之上的组合框架。
- 与 `revit-selection-pickobject-filter` 的区别: PickBox 是交互式取框；本 skill 解决拿到框之后怎么过滤。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定查询语义**：包含于区域 / 与区域相交 / 包含点，三选一；构造对应 Outline（或从 PickBox 结果转换）。
   - 完成标准: 三个过滤器中选定一个，Outline 参数就绪。

2. **评估误报风险**：检查目标图元类别是否含旋转/异形几何（斜墙、斜支撑、弧形构件）。轴对齐规则几何 → bbox 结果直接可用。
   - 完成标准: 明确记录"直接可用"或"需要复核"。
   - 判停条件: 若目标全是轴对齐规则图元（如正交轴网上的矩形柱），跳过步骤 3。

3. **几何级复核**：对 bbox 候选集叠加 ElementIntersectsSolidFilter（SlowFilter，只作用于候选集），剔除误报。
   - 完成标准: 最终结果经过几何验证，无误报残留。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要几何级精确结果且候选集很小——直接上 ElementIntersects* 过滤器，不必先走 bbox。
- 查询语义不是空间关系（按类别、按参数）——用对应过滤器家族。

### 作者在书中警告的失败模式

- 对旋转/异形元素直接信任 bbox 结果——误报是系统性行为，不是偶发 bug。
- 忘记 bbox 过滤器虽是 QuickFilter，但精确复核的 SlowFilter 必须与 QuickFilter 组合使用。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 API 陆续补充了更多空间/几何过滤选项，但"bbox 快筛 + 几何复核"的两段式权衡逻辑未变。

### 容易混淆的邻近方法论

- BoundingBoxIntersectsFilter（图元 bbox 与框相交）与 ElementIntersectsSolidFilter（图元实体与实体相交）语义不同：前者是框级近似，后者是几何级精确——替换时要意识到语义也随之精确化。

---

## 相关 skills

- revit-filter-quick-slow-logical：depends-on——bbox 家族是 QuickFilter 的具体实现，三类策略提供其上的选择框架与组合规则。
- revit-intersects-filter-decisions：composes-with——书主推"bbox 快筛先行、相交过滤器几何复核"两段式；按精度需求也可二选一（对比）。
- revit-selection-pickobject-filter：composes-with——PickBox 产出框，本 skill 负责把框转成 Outline 过滤器输入做空间过滤。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
