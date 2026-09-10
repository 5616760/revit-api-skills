---
name: revit-transaction-hierarchy
description: |
  组织多个模型修改时选择事务层级。何时调用：批量创建/修改后需整体回滚或合并为一个撤销项；事务内局部尝试失败想只撤一小段。何时不调用：单次小修改；不需要回滚语义。Trigger：'批量创建/整组回滚/撤销合并/事务组/嵌套事务'（batch transaction, transaction group, rollback everything, nested transaction）。想在一个事务里再开 Transaction 即应改 SubTransaction 或 TransactionGroup。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.1（约p325-328）
tags: [transaction, sub-transaction, transaction-group, rollback, assimilate]
related_skills:
  - slug: revit-transaction-thread-context
    relation: depends-on
---

# 事务三件套层级与组合

## R — 原文 (Reading)

> Revit API 中与事务相关的主类有三个：Transaction（事务）、SubTransaction（子事务）、TransactionGroup（事务组）。Transaction 是便于 Revit 模型作任意更改所需的上下文。一次只能打开一个事务；不允许嵌套使用。每个事务都须有个名称。……
>
> — 宦国胜, 第5章 5.2.1（约p325-328）

---

## I — 方法论骨架 (Interpretation)

Revit 用三层容器组织"修改模型"的操作，层与层之间是包含关系，各有各的用途：

- **Transaction（事务）**：最小修改单元。所有改模型的代码必须包在一个事务里，事务有名字、一次只能开一个、禁止嵌套。提交（Commit）就生效，回滚（RollBack）就作废。
- **SubTransaction（子事务）**：只能开在已开启的事务内部。它把事务里的一部分操作单独"标出来"，可以只回滚这一部分，不影响外层事务的其他修改。适合"先试试，不行就局部放弃"。
- **TransactionGroup（事务组）**：把多个彼此独立的事务装成一组。结束时有三种选择——提交（组内事务逐个生效）、回滚（把组内**已提交**的事务全部撤销）、融合（Assimilate，把多个事务合并成撤销菜单里的**一个**撤销项）。

核心判断：想让用户"一键撤销整批操作"或"失败时整组作废"，用 TransactionGroup；只想在单个事务内部放弃局部尝试，用 SubTransaction；普通单次修改，用 Transaction。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: MEP 管道创建必须包裹事务
- **问题**: 第4章 4.3 中 MEP 管道创建（Autodesk.Revit.DB.Plumbing 等）也是模型修改。
- **方法论的使用**: 按第2章 2.5"所有图元修改操作都需要 Transaction 包裹"的规则，把管道路径创建放进事务。
- **结论**: 事务是"能否写模型"的硬门槛，不分专业模块。
- **结果**: 漏包事务的代码在运行时抛 ModificationOutsideTransactionException。

### 案例 2: Assimilate() 合并撤销项
- **问题**: 一个命令流程里连续提交了多个事务，用户撤销时想一步回到最初，而不是点 N 次 Ctrl+Z。
- **方法论的使用**: 第5章 5.2.1.3 代码 5-11 演示用 TransactionGroup 包住多个 Transaction，结束时调 Assimilate()。
- **结论**: 融合后多个事务在撤销菜单里合并成带组名的单一撤销项。
- **结果**: 用户一步撤销整批操作，交互体验与"原子操作"一致。

### 案例 3: 批量创建+分析+整组回滚
- **问题**: 先创建一批图元，再基于它们做批量分析，失败要全部撤销。
- **方法论的使用**: V2 预测场景——TransactionGroup 包多个 Transaction，每个 Transaction 建一组图元；分析失败时调 TransactionGroup.RollBack()。
- **结论**: 三层结构允许"中间已提交、顶层仍可整组回滚"。
- **结果**: 分析失败不留任何残留修改；只想局部尝试失败不影响外层时，改用 SubTransaction 局部 RollBack。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批量生成大量图元（如一整层楼的门、几百根梁），要求"失败就全部还原"或"撤销时一步到位"。
2. 一个事务内部做多步修改，其中某一步想"试试，不行就放弃这一小段"，其他修改保留。
3. 插件流程由多个独立事务组成，希望结束失败时把已提交的事务也一并撤销。
4. 用户抱怨"撤销要按很多次"——需要把多次事务合并成一个撤销项。

### 语言信号 (用户的话里出现这些就应激活)

- "批量创建这么多图元，失败了能全部撤销吗？" / "rollback all if anything fails"
- "怎么把这几步操作合并成一个撤销项？" / "merge into one undo step"
- "嵌套事务怎么写？" / "nested transaction"
- "事务组、子事务、临时事务有什么区别？" / "transaction group vs sub transaction"
- "想撤销一半的修改" / "undo only part of my changes"

### 与相邻 skill 的区分

- 与 `revit-transaction-thread-context` 的区别: 本 skill 讲事务的层级结构（Transaction/SubTransaction/TransactionGroup），thread-context 讲启动事务的上下文门槛；先过上下文关，再谈层级。
- 与 `revit-regenerate-failure-rollback`（revit-regenerate-failure-rollback）：那个讲 Regenerate 失败后必须回滚的恢复契约；本 skill 讲层级结构本身。
- 与 `revit-failure-handling-options`（revit-failure-handling-options）：那是事务收尾时的故障处理配置；本 skill 是事务结构组织，两者互补不冲突。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定需要的层级**：先问"这次要写模型吗？粒度要求是什么？"
   - 只有一步小修改 → 单 Transaction。
   - 事务内想局部放弃 → 加 SubTransaction。
   - 多个独立事务要整体回滚/合并撤销 → 包 TransactionGroup。
   - 完成标准: 已明确选一层或两层组合，并说出理由。

2. **按层级打开与关闭**：
   - Transaction: `new Transaction(doc, "name").Start()` … `Commit()` / `RollBack()`。
   - SubTransaction: 只能在已开事务内 `new SubTransaction(doc).Start()` … `Commit()` / `RollBack()`。
   - TransactionGroup: `new TransactionGroup(doc, "name").Start()` 包住多个事务，结束选 `Commit()` / `RollBack()` / `Assimilate()`。
   - 判停条件: 若想在一个 Transaction 内再开 Transaction（嵌套），**停下**，改外层为 TransactionGroup 或内层为 SubTransaction。

3. **收尾校验**：
   - 检查每个 Start 都有对应的 Commit/RollBack（成对出现）。
   - 检查整组失败路径上调用了 TransactionGroup.RollBack()。
   - 用 `TransactionStatus` 验证最终状态是 Committed/RolledBack（而非 Pending/Error）。
   - 完成标准: 所有事务/子事务/组都正确闭合，失败路径可整组撤销。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单个简单修改还套三层结构——过度设计，撤销菜单反而混乱。
- 不需要回滚/合并语义的纯只读操作（读几何、查询参数）——不该开任何事务。
- 线程外或非模态对话框里想开事务组——先解决线程上下文（见 `revit-transaction-thread-context`）。

### 作者在书中警告的失败模式

- **一次只能打开一个事务，不允许嵌套**——在事务内再 new Transaction 会抛异常（或状态 Error）。
- **SubTransaction 只能在已开启的事务内创建**——事务外建子事务非法。
- **TransactionGroup 回滚会撤销组内已提交的事务**——别以为 Commit 过就"安全"了，组级回滚会连坐。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 API / .NET 4.0：事务状态机常量与异常类型可能略有出入，新版 API 中若干事务相关方法（如 Commit 失败时的 FailureProcessingResult 处理）细节需查当前版 SDK 文档核对。
- 未深入讨论"长事务对中央文件工作共享的影响"——大 TransactionGroup 长时间持有锁会阻塞协作者。

### 容易混淆的邻近方法论

- `SubTransaction` vs `TransactionGroup`：前者管"事务内部的局部撤销"，后者管"多个事务的整体撤销/合并"，范围方向相反。
- `Assimilate()`（合并为撤销一项）vs 普通 `Commit()`（组内事务保持各自撤销项）：撤销粒度的选择，不是性能选择。

---

## 相关 skills

- revit-transaction-thread-context：depends-on——本 skill 讲事务层级结构，前提是事务能在受支持上下文（主线程/命令/事件）中启动，thread-context 提供该前提判断。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
