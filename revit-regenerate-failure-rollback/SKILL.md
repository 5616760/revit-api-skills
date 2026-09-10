---
name: revit-regenerate-failure-rollback
description: |
  Regenerate() 抛 RegenerationFailedException 或事务陷入半完成状态时使用。规则：失败必须回滚当前事务或子事务，不能忽略。何时调用：Regenerate 抛异常；读几何发现不一致。何时不调用：Regenerate 未失败。Trigger：'regeneration failed / RegenerationFailedException / 事务半完成 / 修改没回滚'。Regenerate 失败即把本次修改整体作废并回滚。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.4（约p331）
tags: [regenerate, rollback, failure-recovery, counter-example, transaction]
related_skills:
  - slug: revit-transaction-hierarchy
    relation: depends-on
  - slug: revit-temporary-transaction-analysis
    relation: contrasts-with
---

# 反例：Document.Regenerate() 失败抛异常时未回滚会导致"半完成"状态

## R — 原文 (Reading)

> 应当指出的是，Regenerate() 方法可能会失败，在这种情况下，将引发 RegenerationFailedException。如果发生这种情况，则需通过回滚当前事务或子事务来对文件的更改作回滚处理。
>
> — 宦国胜, 第5章 5.2.4（约p331）

---

## I — 方法论骨架 (Interpretation)

Regenerate() 不是"刷新一下 UI"那么简单——它要重算参数化关系、传播修改、重建几何，这个过程中可能失败（比如参数取值导致族不可再生、图元引用悬空），失败时抛 `RegenerationFailedException`。

失败后最危险的事是**忘了回滚**：事务里此前已经应用的修改留在内存里，形成一个"半完成/中间态"——没完整重生成、也没持久化，模型处于不一致状态。如果继续在它上面做后续操作，会基于坏数据计算，可能产生悬空引用、错误结果，甚至把不一致状态带到下一个事务。

因此这本书给出一个明确的**恢复契约**：捕获 RegenerationFailedException 后，用 RollBack 回滚当前事务或子事务，把这次"修改尝试"整体作废、当作无事发生。这也解释了为什么临时事务分析技术（revit-temporary-transaction-analysis）总是以 RollBack 收尾——"故意让重生成/修改失败、然后撤销"本身就是一种无副作用的分析手段。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: Regenerate 失败未回滚的中间态
- **问题**: V2 预测场景——Regenerate() 失败且忘了回滚，文件处于什么状态？
- **方法论的使用**: 按恢复契约分析——部分修改已应用到内存但未完整重生成。
- **结论**: 事务处于"中间态"，继续操作基于不一致模型，可能出现图元悬空引用。
- **结果**: 必须 catch RegenerationFailedException 后显式 RollBack 子事务或父事务，把 Regenerate 失败当成"本次修改作废"的触发器。

### 案例 2: 临时事务分析的收尾方式
- **问题**: 临时事务技术（revit-temporary-transaction-analysis）里删除图元、Regenerate、提取几何后怎么收尾？
- **方法论的使用**: 反正要撤销，把 RollBack 当作标准结束动作。
- **结论**: 临时事务/分析型事务天然用 RollBack 收尾——"成功"的标志就是"模型回到原状"。
- **结果**: 任何 Regenerate 失败都不留残渣，分析结果可靠。

### 案例 3: 提交时的自动重生成失败
- **问题**: 第5章 5.2.4 及附录B FAQ 指出提交事务时也会自动重生成——这同样可能失败。
- **方法论的使用**: 把"提交"也理解为一次重生成触发点。
- **结论**: 提交失败同样不能当成功，要处理故障并决定回滚。
- **结果**: 批量脚本里对每个事务提交做状态检查，失败即回滚。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件在事务内调 Regenerate 后程序抛 RegenerationFailedException，用户报告崩溃/状态错乱。
2. 改了参数、Regenerate、然后后续步骤读到的数据不对，怀疑是"中间态"污染。
3. 用户问"Regenerate 失败会发生什么？需要处理吗？"
4. 批量脚本里某次提交失败后，后面的操作全部基于坏状态，需要理解回滚的必要性。

### 语言信号 (用户的话里出现这些就应激活)

- "Regenerate 抛异常了，要不要处理？" / "Regenerate throws an exception"
- "RegenerationFailedException 是什么意思" / "what is RegenerationFailedException"
- "事务回滚 / 半完成状态 / 修改没撤销" / "half-committed state, changes not rolled back"
- "重生成失败后还能继续吗？" / "can I continue after regeneration failure"
- "提交时自动重生成也会失败吗" / "does commit-time regeneration fail"

### 与相邻 skill 的区分

- 与 `revit-transaction-hierarchy` 的区别: 本 skill 讲 Regenerate 失败后回滚当前事务/子事务的恢复契约，transaction-hierarchy 讲事务层级结构本身；回滚动作作用于层级中的某个事务。
- 与 `revit-temporary-transaction-analysis` 的区别: 临时事务是"主动 RollBack 当分析武器"，本 skill 是"失败必须回滚"的恢复契约——一个主动、一个被动。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **用 try-catch 包住 Regenerate()**：在事务内调用 Regenerate()/AutoJoinElements()，并捕获 `RegenerationFailedException`。
   - 完成标准: 确认 Regenerate 调用处于事务内，且异常类型已捕获（可 catch `Autodesk.Revit.Exceptions.RegenerationFailedException`）。

2. **失败即回滚**：catch 分支里调用当前事务（或子事务）的 `RollBack()`，把本次修改整体撤销。
   - 完成标准: RollBack 成功，`TransactionStatus` 变为 RolledBack；或子事务回滚后外层事务可继续。
   - 判停条件: 若没有开启事务/子事务可供回滚（比如在事务外误调 Regenerate），先停下，修复代码结构，不要继续操作。

3. **恢复正常流程**：回滚后按"本次操作作废"处理——返回错误提示、跳到下一个迭代、或重试整段逻辑。
   - 完成标准: 模型回到该事务开始前的状态；后续操作基于干净数据；无悬空引用。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- Regenerate 没有失败——不用每次调用都假设失败，正常路径正常走。
- 提交时自动重生成失败——那是事务提交阶段的故障处理问题（走故障处理流水线），不完全等同事务内手动 Regenerate。

### 作者在书中警告的失败模式

- **Regenerate() 可能失败并抛 RegenerationFailedException**——必须预期它，不能裸调。
- **失败后不回滚 = 半完成状态**——部分修改残留内存，后续基于不一致模型操作，产生悬空引用。
- **Regenerate()/AutoJoinElements() 只能在开启的事务内部调用**——事务外调用本身也是异常路径。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版中 RegenerationFailedException 的子类与触发面有调整（如族重生成、几何运算失败的细分异常），catch 范围应核对当前 SDK。
- 未讨论"Regenerate 失败后能否只撤销一部分"的更多细节——书中只给了"回滚当前事务或子事务"两种手段，粒度边界要靠工程实践补充。

### 容易混淆的邻近方法论

- Regenerate 失败回滚 vs 事务提交失败回滚：前者是手动触发后的恢复；后者是提交阶段由故障处理系统（Preprocessor/Processor）裁决，别混为一谈。
- "回滚" vs "撤销（Ctrl+Z）"：RollBack 是 API 级事务回滚，影响的是当前事务的修改；撤销菜单是用户级操作，两者不在一个层面。

---

## 相关 skills

- revit-transaction-hierarchy：depends-on——本 skill 讲 Regenerate 失败后回滚当前事务/子事务，前提是理解事务/子事务/事务组的层级结构与闭合规则。
- revit-temporary-transaction-analysis：contrasts-with——本 skill 是被动的"失败必须回滚"，临时事务是主动的"故意 RollBack 做分析"，两种回滚动机互为对照。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
