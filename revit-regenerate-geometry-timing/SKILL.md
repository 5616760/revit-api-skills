---
name: revit-regenerate-geometry-timing
description: |
  创建/修改图元后立即读几何或分析模型时使用。何时调用：刚 new 完图元要测体积/厚度；改参数后依赖新几何计算。何时不调用：只改不读；提交后读。Trigger：'geometry is null after create / 读不到几何 / analytical model not available / 什么时候能读几何'。规则：Regenerate()/AutoJoinElements() 只能在开启事务内调用，提交时自动重生成。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.4（约p331）
tags: [regenerate, geometry, analytical-model, timing, transaction]
related_skills:
  - slug: revit-regenerate-failure-rollback
    relation: composes-with
  - slug: revit-regenerate-cost
    relation: composes-with
---

# 事务提交后填充 Geometry / AnalyticalModel 的时序约束

## R — 原文 (Reading)

> 在创建新图元或修改图元之后，传播整个模型的变化需要图元重生成和自动连接。若没有重生成（以及自动连接，当相关时），图元的 Geometry 属性和 AnalyticalModel 要么是不可获取的（在创建新图元的情况下），要么可能是无效的。……
>
> — 宦国胜, 第5章 5.2.4（约p331）

---

## I — 方法论骨架 (Interpretation)

Revit 不是"改完立即生效"的即时一致性模型，而是**延迟重生成**：

- 你在事务里创建/修改图元后，几何体和分析模型**不会立刻更新**。此刻去读 `Geometry` 属性——新建的图元返回 null/不可获取，改过的图元返回旧值/无效值。
- 重生成只发生在三个时机之一：① 事务成功提交时（自动触发一次）；② 手动调 `Document.Regenerate()`；③ 手动调 `Document.AutoJoinElements()`。
- ②和③**只能在开启的事务内部调用**，事务外调用直接抛异常。

所以"创建 → 立即读几何 → 继续计算"这个直觉流程在 Revit 里会踩空。正确节奏是：**创建/修改 → （事务内）Regenerate() → 读几何 → 计算**，并把 Regenerate 可能抛 `RegenerationFailedException` 一并纳入错误处理。这也是为什么"创建→分析→再改"的循环要么拆成多个事务、要么在事务内强制重生成。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 新建图元后取几何得到 null
- **问题**: 在事务里创建一面墙，立刻想测它的厚度/体积，得到 null。
- **方法论的使用**: 按时序约束——事务内直接访问 Geometry 在重生成发生前是无效的。
- **结论**: 必须先 `Document.Regenerate()` 强制重生成，再取几何。
- **结果**: 拿到有效几何，但必须 catch RegenerationFailedException 并回滚（联动 revit-regenerate-failure-rollback）。

### 案例 2: GetOriginalGeometry 与当前几何差异
- **问题**: 第3章 3.7.5 里要区分"图元原始几何"与"当前几何"。
- **方法论的使用**: 意识到几何是分"时点"的——原始几何在重生成前可能仍是旧值。
- **结论**: 取几何前先确认你处于哪个时点、是否需要触发重生成。
- **结果**: 分析代码避免了"读到过期几何"的错误结论。

### 案例 3: DocumentChanged 只读通知
- **问题**: 想在文档变化事件里立刻读新几何做联动。
- **方法论的使用**: 第5章 5.2.2 明确 DocumentChanged 是只读通知。
- **结论**: 事件里不能开事务、不能改模型，只能记录"变化已发生"。
- **结果**: 联动逻辑改为排入后续事务执行，而不是在事件里即时读几何。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件"建墙后立刻量体积/算数量"得到 null 或 0，用户报 bug。
2. 改了一个图元的参数，后面立刻依赖新几何做偏移/碰撞检查，结果用了旧几何。
3. 想要分析模型（AnalyticalModel）做结构计算，却拿不到数据。
4. 事务提交后发现几何才"变对"，需要理解"为什么要拆成两个事务"。

### 语言信号 (用户的话里出现这些就应激活)

- "刚创建的图元 Geometry 是空的/拿不到" / "geometry is null after creation"
- "改了参数但几何没变/读到的是旧数据" / "stale geometry after parameter change"
- "AnalyticalModel 取不到" / "analytical model not available"
- "什么时候才能读几何？" / "when can I read the geometry"
- "Regenerate 有什么用、什么时候调？" / "when to call Regenerate"

### 与相邻 skill 的区分

- 与 `revit-regenerate-failure-rollback`（revit-regenerate-failure-rollback）：那个是"Regenerate 失败后必须回滚"；本 skill 是"为什么提交前读不到几何、何时重生成"。
- 与 `revit-regenerate-cost`（revit-regenerate-cost）：那个劝你"别频繁调 Regenerate"；本 skill 说"该调时要调"。两者合并成一条决策：只在"需要立即读新几何"时事务内调一次。
- 与 `revit-temporary-transaction-analysis`（revit-temporary-transaction-analysis）：那个也用 Regenerate，但目的是故意改模型后提取假设几何并回滚；本 skill 是常规取几何的时序规则。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认你是否在事务内**：检查代码路径——要读新建图元的几何，必须在一个开启的 Transaction/SubTransaction 内部。
   - 完成标准: 确认当前处于事务内；若不在，先包事务（只读场景则可直接提交后读）。
   - 判停条件: 若方案是"读几何但完全不需要最新值"，跳过 Regenerate，直接走提交后再读。

2. **创建/修改图元后显式触发重生成**：`doc.Regenerate()`（需要自动连接时用 `AutoJoinElements()`），放在事务内、取几何之前。
   - 完成标准: Regenerate 调用成功返回，未抛异常。

3. **读取几何并处理失败**：
   - 读 `Element.Geometry` 或 `AnalyticalModel`，校验非 null 且有效。
   - 用 try-catch 包住 Regenerate，捕获 `RegenerationFailedException` 后 RollBack 当前事务/子事务。
   - 完成标准: 拿到的几何是新值；Regenerate 失败时事务已回滚，无"半完成"残留。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只是批量改参数、不需要中间读几何——提交后系统自动重生成一次，别手动调（见 `revit-regenerate-cost`）。
- 在事件回调（如 DocumentChanged）里想开事务调 Regenerate——事件是只读通知，改模型/开事务受限。

### 作者在书中警告的失败模式

- **Regenerate() 可能抛 RegenerationFailedException**——不改、不 catch 会留下半完成状态（对应 revit-regenerate-failure-rollback 的完整反例）。
- **Regenerate()/AutoJoinElements() 只能在开启的事务内部调用**——事务外调必抛异常。
- **事务内直接访问新建图元的 Geometry 可能返回 null**——不是"偶尔抽风"，是机制如此。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，`AutoJoinElements` 的行为在新版（如 2020+ 的元素连接改进）有差异；几何服务（GeometryAPI/隐式几何）在后续版本有独立入口。
- 未覆盖"族实例几何在族内/项目内的缓存差异"——某些情况下还需处理 `GeometryInstance` 层级。
- 未讨论 Regenerate 在超大模型上的耗时问题——性能要点见 revit-regenerate-cost。

### 容易混淆的邻近方法论

- `Document.Regenerate()` vs 提交时自动重生成：前者是"我马上要读新几何"，后者是"系统收尾"——别都当一回事反复调。
- `Geometry` vs `AnalyticalModel`：两者都受同一时序约束，别只处理几何而漏了分析模型。

---

## 相关 skills

- revit-regenerate-failure-rollback：composes-with——本 skill 讲何时重生成、为何提交前读不到新几何，rollback 讲 Regenerate 失败后必须回滚，时序规则与失败恢复配套。
- revit-regenerate-cost：composes-with——本 skill 说"该调时要调"（立即读新几何），regenerate-cost 劝"别频繁调"（性能），合并成一条决策。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
