---
name: revit-temporary-transaction-analysis
description: |
  做假设分析（what-if）时使用：临时改模型、提取信息、再回滚。何时调用：提取开洞前完整几何；测试删掉某图元的结果；临时 Solid 分析。何时不调用：修改要保留；纯只读查询。Trigger：'开洞前的几何/临时删除算一下/假设分析/what-if'（temporary transaction, rollback after analysis）。核心四步：Start→Delete/修改→Regenerate→RollBack，必须 Regenerate 才拿到新几何。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.5（约p348）
tags: [temporary-transaction, what-if, rollback, geometry-analysis, pattern]
related_skills:
  - slug: revit-transaction-hierarchy
    relation: depends-on
  - slug: revit-regenerate-geometry-timing
    relation: depends-on
---

# 临时事务（Temporary Transactions）分析技术

## R — 原文 (Reading)

> 使用临时事务可用于某些类型的分析。例如，若应用程序要提取洞口切割之前的墙或其他对象几何属性，则应连同 Document.Delete() 一起使用临时事务。当应用程序删除切割目标图元的那个图元时，被切割图元的几何形状会恢复到其原始状态……
>
> — 宦国胜, 第5章 5.2.5 临时事务（约p348）

---

## I — 方法论骨架 (Interpretation)

正常情况下事务以 Commit（保留）或 RollBack（撤销）结束。**临时事务（Temporary Transaction）**把这个模型反过来用：**故意只做"临时修改 + RollBack 撤销"，把事务当成一个可逆的实验场**。

核心套路是四步：开事务 → 在内存里做修改（如 `Document.Delete` 删掉洞口图元）→ 调 `Document.Regenerate()` 让几何重算 → 提取你想要的信息（比如墙体开洞前的完整几何）→ 最后 `Transaction.RollBack()` 把一切撤销，模型回到原状。

它的威力来自 RollBack 的**无副作用**：删除、重生成、几何变化全部撤销，文件里不留任何痕迹。所以"提取开洞前的几何"这种在已建成模型上不可能直接做到的事，用临时事务就能办到——你临时把洞口删了，墙就"变回"完整形状，量完再撤销。

这个技术同样适用于 SubTransaction，也配合临时 Solid 分析（第3章 3.7）使用。本质是"假设分析（what-if）"的通用实现：任何"如果我改一下，会得到什么"的试算，都可以用临时事务做，前提是你愿意承受一次 Regenerate 的开销（临时事务内必须有 Regenerate，否则几何不更新）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 提取洞口切割前的墙体几何
- **问题**: 模型里墙已被窗/门开洞，想测量墙体的完整几何（开洞前），模型已建好无法回退。
- **方法论的使用**: 用临时事务 + Document.Delete() 临时删除洞口图元，触发重生成。
- **结论**: 被切割图元的几何恢复到原始（完整）状态。
- **结果**: 提取完整几何，然后 RollBack，文件回到原状（V2 预测场景验证）。

### 案例 2: 临时 Solid 分析的配套
- **问题**: 第3章 3.7 几何提取中要对 Solid 做布尔/交集分析。
- **方法论的使用**: 把"临时修改→提取→撤销"与临时 Solid 结合。
- **结论**: 假设分析工具箱 = 临时事务 + 临时 Solid。
- **结果**: 分析代码不污染模型，可反复运行。

### 案例 3: SubTransaction 版本的局部试算
- **问题**: 不想开整个事务，只想在已开启的大事务内部对一小段做假设验证。
- **方法论的使用**: 临时事务技术同样适用于 SubTransaction——局部 Start → 修改 → RollBack。
- **结论**: 局部试算失败不影响外层事务。
- **结果**: 在复杂命令中实现"验证一下再决定"的分支逻辑。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要量"被开洞前"的墙/板/梁几何，而模型已经是最终形态。
2. 想测试"删掉这个支撑构件，上面楼层会不会掉"这类结构假设。
3. 批量分析脚本要对同一模型反复做不同假设的几何计算。
4. 命令里要做"试算验证再决定是否真的修改"的分支。

### 语言信号 (用户的话里出现这些就应激活)

- "开洞前的完整几何怎么拿？" / "geometry before the opening was cut"
- "临时删掉这个图元算一下" / "temporarily delete this element and compute"
- "假设分析 / 试算不落地 / what-if" / "what-if analysis, hypothetical check"
- "RollBack 之后模型会还原吗？" / "does rollback restore everything"
- "临时事务 / temporary transaction" / "temporary transaction analysis"

### 与相邻 skill 的区分

- 与 `revit-transaction-hierarchy`（revit-transaction-hierarchy）：那个管"如何组织永久性事务结构"；本 skill 是"故意用 RollBack 做分析"，方向相反（分析而非落库）。
- 与 `revit-regenerate-geometry-timing`（revit-regenerate-geometry-timing）：临时事务内同样依赖 Regenerate 才拿到新几何；本 skill 是把该时序规则当作工具链一环。
- 与 `revit-regenerate-failure-rollback`（revit-regenerate-failure-rollback）：那个是"失败必须回滚"的恢复契约；本 skill 是"主动回滚当武器"。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **开临时事务并做假设修改**：
   - `new Transaction(doc, "temp analysis").Start()`。
   - 做假设修改：如 `doc.Delete(openingElementId)`、改参数、移动图元等。
   - 完成标准: 修改已应用（内存中），未提交。
   - 判停条件: 若只是只读查询（不需要改任何东西），**停下**，不需要事务，直接读几何。

2. **重生成并提取信息**：
   - 事务内调 `doc.Regenerate()`（或 AutoJoinElements）强制几何更新。
   - 提取目标图元的 Geometry / AnalyticalModel / 自定义计算值。
   - 完成标准: 提取到的是"假设状态下"的新几何，非 null 有效。
   - 判停条件: 若 Regenerate 抛 RegenerationFailedException，直接进入步骤 3 的 RollBack（反正要撤销）。

3. **RollBack 收尾**：
   - 无论提取成功还是失败，调 `transaction.RollBack()`。
   - 完成后校验：原模型无残留修改，`TransactionStatus` 为 RolledBack。
   - 把提取到的数据返回给上层逻辑使用。
   - 完成标准: 模型与事务开始前完全一致；数据已安全传出。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此技能

- 修改要保留——必须 Commit，不能贪方便用临时事务（数据会丢）。
- 纯只读查询——不需要事务，开事务反而多一次不必要的开销。
- 修改集合非常大或 Regenerate 极慢的场景——临时事务强制重生成，开销必须计入。

### 作者在书中警告的失败模式

- **临时事务内忘调 Regenerate**——几何不更新，提取到的还是旧值（时序约束失效）。
- **临时事务结束后忘了 RollBack**——修改意外保留，污染模型（最严重的反模式）。
- **Regenerate 可能失败**——虽然临时事务反正要撤销，但没 catch 会让异常冒泡打断流程。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：临时事务与 `Document.Delete` 配合的行为在新版里对"族实例/编组/链接"的处理有差异，复杂删除应先在测试文件验证。
- 未讨论临时事务在性能上的代价（强制重生成），大批量假设分析要注意速度。
- 未深入"撤销后外部引用/视图刷新"的副作用——RollBack 不等于 UI 状态也回退。

### 容易混淆的邻近方法论

- 临时事务（RollBack 收尾） vs 正常事务（Commit 收尾）：唯一区别是结束动作，但意图完全不同——分析 vs 落地。
- 临时事务 vs SubTransaction 局部试算：前者独立开整个事务，后者嵌在已开启事务内部——别混用导致嵌套错误。

---

## 相关 skills

- revit-transaction-hierarchy：depends-on——本 skill 用事务做假设分析，前提是先理解事务层级结构与 RollBack 语义，才能安全地把回滚当分析手段。
- revit-regenerate-geometry-timing：depends-on——临时事务内必须 Regenerate 才拿到新几何，本 skill 依赖该时序规则作为工具链一环。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
