---
name: revit-failure-handling-options
description: |
  按事务定制故障处理策略时使用。五开关：ClearAfterRollback、DelayedMiniWarnings、ForcedModalHandling、SetFailuresPreprocessor、SetTransactionFinalizer。何时调用：批处理关弹窗、延迟警告、清空缓冲、挂钩子。何时不调用：接受默认交互。Trigger：'警告弹窗太烦/延迟警告/回滚后清空警告'（suppress warnings, delayed mini warnings）。非模态处理后事务可能仍 Pending，需轮询 GetStatus。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.3（约p330）
tags: [failure-handling, warnings, modal, transaction-finalizer, batch]
related_skills:
  - slug: revit-failures-processing-steps
    relation: depends-on
  - slug: revit-failure-options-get-set
    relation: depends-on
  - slug: revit-failures-processor-global
    relation: contrasts-with
  - slug: revit-failures-preprocessor
    relation: composes-with
---

# 故障处理选项（FailureHandlingOptions）配置框架

## R — 原文 (Reading)

> 1. 回滚后清除（ClearAfterRollback）：此选项控制是否在事务回滚后清除所有警告。默认值为 false。
> 2. 延迟小警告（DelayedMiniWarnings）：如果有"小"警告的话，此选项控制它是否在当前结束的事务未显示，或是延迟到下一个事务结束时显示。……
>
> — 宦国胜, 第5章 5.2.3（约p330）

---

## I — 方法论骨架 (Interpretation)

事务结束时 Revit 默认会弹故障对话框。`FailureHandlingOptions` 是给"这个事务"定制故障处理策略的五个开关（配置对象不能 new，必须 Get+Set 写回，见 revit-failure-options-get-set）：

1. **ClearAfterRollback**：回滚后要不要清空警告缓冲。默认 false（不清）。批处理里常开 true，避免旧警告污染下次判断。
2. **DelayedMiniWarnings**：小警告（如"墙没连接"）是当前事务结束立刻显示，还是延迟到下一个事务结束时显示。默认 false。想"攒一批再提示"就开 true。
3. **ForcedModalHandling**：最终故障走模态弹窗（阻塞等用户）还是非模态（不阻塞）。默认 true。批处理关掉它 = 弹窗消失，但**代价是 Commit/RollBack 返回时事务可能仍是 Pending 状态**，必须轮询 `GetStatus()` 等它结束。
4. **SetFailuresPreprocessor**：给事务挂故障预处理器（IFailuresPreprocessor），故障发生时（且仅当发生时）在事务结束时被调用，可检查/尝试解决故障。
5. **SetTransactionFinalizer**：挂事务终结器，事务结束时执行自定义收尾逻辑。

这五个开关是"配置"，真正的处理逻辑由故障处理三步骤流水线执行（revit-failures-processing-steps~f13）。配置层负责"要不要弹窗、警告去哪、挂不挂钩子"，流水线负责"怎么处理"。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批处理关弹窗、延迟警告、清缓冲
- **问题**: V2 预测场景——批处理插件要连续跑 100 个事务，不希望每步的无关警告弹窗打断，也不希望警告堆积。
- **方法论的使用**: 组合配置——SetForcedModalHandling(false) 关模态弹窗 + SetDelayedMiniWarnings(true) 把连续事务的小警告推迟到批次末一次性处理 + SetClearAfterRollback(true) 回滚后清空警告缓冲。
- **结论**: 三个开关组合实现"安静批处理"。
- **结果**: 批处理无弹窗、无警告堆积，批次末统一看到摘要。

### 案例 2: 挂事务终结器
- **问题**: 想在每个事务结束时自动记录耗时/日志。
- **方法论的使用**: SetTransactionFinalizer 挂自定义 IT TransactionFinalizer 接口。
- **结论**: 终结器在事务结束时执行自定义操作。
- **结果**: 收尾逻辑（日志、统计）不再散落各处。

### 案例 3: 非模态处理下的 Pending 态
- **问题**: ForcedModalHandling(false) 后 Commit 返回，事务真结束了吗？
- **方法论的使用**: 注意非模态处理时 Commit/RollBack 返回后事务可能仍是 Pending。
- **结论**: "Commit 返回 ≠ 事务完成"。
- **结果**: 需要轮询 GetStatus() 确认事务真正结束（或由故障流水线裁决）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批处理脚本被每步的警告弹窗打断，用户体验差。
2. 连续事务的小警告想合并到批次末一次性显示。
3. 事务回滚后想清空警告缓冲，避免旧警告干扰判断。
4. 想给特定事务挂预处理/终结器钩子。

### 语言信号 (用户的话里出现这些就应激活)

- "批处理时怎么关掉警告弹窗？" / "suppress warning dialogs in batch"
- "警告别一个个弹，攒一起提示" / "delay mini warnings until the end"
- "回滚后警告还在，怎么清空？" / "clear warnings after rollback"
- "给事务挂终结器/预处理器" / "set a transaction finalizer or preprocessor"
- "ForcedModalHandling 关了之后事务什么状态？" / "transaction status after non-modal handling"

### 与相邻 skill 的区分

本 skill 与 `revit-failure-options-get-set`、`revit-failures-processing-steps` 区分：前两者分别是配置对象的读写手筋与处理流水线框架，本 skill 聚焦五个开关各自语义。`revit-failures-processor-global` 是全会话级处理器，与事务级 options 配置作用域相反；`revit-failures-preprocessor` 通过本 skill 的 SetFailuresPreprocessor 挂载，接口行为见该 skill。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确批处理需求**：列出"哪些弹窗必须关、哪些警告要延迟、回滚后要不要清空"。
   - 完成标准: 五个开关各自的取值已确定（含默认值理由）。

2. **Get+Set 配置并写回**：
   - `var opts = transaction.GetFailureHandlingOptions();`
   - 依次调用需要的 Set 方法，最后 `transaction.SetFailureHandlingOptions(opts);`
   - 完成标准: 配置写回成功。

3. **处理非模态后的 Pending 态**：
   - 若 ForcedModalHandling(false)：Commit/RollBack 后轮询 `transaction.GetStatus()`，直到状态离开 Pending。
   - 对每个事务独立配置（不要跨事务共享 options 实例）。
   - 完成标准: 批处理全程无意外弹窗；每个事务最终状态明确（Committed/RolledBack），无悬挂 Pending。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 接受默认故障交互的单次操作——默认配置即可，别过度配置。
- 只读操作（无事务）——没有可配置对象。
- 需要全局改变所有会话错误行为——那是全局 IFailuresProcessor 的职责，不是每个事务配置 options。

### 作者在书中警告的失败模式

- **ForcedModalHandling(false) 后 Commit 返回 ≠ 事务完成**——可能 Pending，不轮询会基于未决状态继续操作。
- **DelayedMiniWarnings 依赖"下一个事务"**——批次结束后最后一个事务的延迟警告仍要处理，别漏。
- **ClearAfterRollback 默认 false**——不主动清空，警告缓冲会累积，可能影响后续判断。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 FailureHandlingOptions 的默认值与个别 setter 行为有调整（如 DelayedMiniWarnings 的显示策略），应核对当前 SDK。
- 未展开"警告缓冲的容量/生命周期"细节——极端批处理下缓冲可能很大。

### 容易混淆的邻近方法论

- `ForcedModalHandling(false)`（非模态处理） vs 模态处理：非模态不阻塞 UI，但引入 Pending 轮询负担——不是"免弹窗免费午餐"。
- 事务级 options vs 全局处理器：前者逐事务配置，后者接管全会话——作用域不同，别互相替代。

---

## 相关 skills

- revit-failures-processing-steps（depends-on）：五开关是配置层，真正处理由三步流水线执行，本 skill 依赖其框架。
- revit-failure-options-get-set（depends-on）：配置对象必须 Get+Set 写回，本 skill 讲五个开关的语义。
- revit-failures-processor-global（contrasts-with）：事务级 options 配置 vs 全局处理器接管全会话，作用域不同。
- revit-failures-preprocessor（composes-with）：SetFailuresPreprocessor 只是挂钩子，预处理器接口行为见该 skill。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
