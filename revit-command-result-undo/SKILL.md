---
name: revit-command-result-undo
description: |
  当问'返回 Failed 会不会撤销修改'、或命令批量改图元中途失败担心一致时调用。Succeeded=保留修改；Failed=自动逆转全部更改+Error 对话框；Cancelled=逆转全部更改+
  Warning 对话框。Automatic 模式下返回值直接驱动回滚，无需写撤销代码。不适用：Manual 精细回滚。trigger：Result 返回值、自动撤销、回滚、auto rollback、
  undo changes。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p034
tags: [external-command, result-enum, transaction-rollback, undo]
related_skills:
  - slug: revit-transaction-hierarchy
    relation: contrasts-with
---

# 外部命令返回值语义与自动撤销行为

## R — 原文 (Reading)

> 返回结果指明命令执行失败、成功或是已被用户取消，见表 1-2。若未成功，则 Revit 会逆转外部命令所作的更改。Failed：外部命令未能完成任务，Revit 逆转外部命令所执行的操作，若已设置消息参数则显示 Error 对话框。
>
> — 宦国胜, 第1章 1.3.2 / 表 1-2（约 p034）

---

## I — 方法论骨架 (Interpretation)

`Result` 不只是"成功/失败标志"，它是**事务的开关**——返回值直接决定本次命令对模型的所有修改是保留还是抹掉：

- `Succeeded`：保留全部修改。
- `Failed`：**自动逆转**命令期间对模型的一切更改（配合 Automatic 事务即整体回滚），并弹 Error 对话框。
- `Cancelled`：同样**自动逆转**一切更改，弹的是 Warning（语气更轻）对话框。

两条推论很实用：

1. **不需要手动写撤销代码**。Automatic 模式下，你只要在失败路径返回 `Result.Failed`，Revit 就会把本次 Execute 里所有 Create/Set/Delete 一并回滚——"改了 5 个图元第 3 个失败"的一致性由返回值保证。
2. **Failed 和 Cancelled 的效果都含回滚**，区别只在对话框类型和语义：Failed=没干成，Cancelled=用户中途退出。

注意：这个"返回值驱动事务"的机制在 `TransactionMode.Automatic` 下最强；`Manual` 模式下事务由你管理，返回值仍会触发外部事务组兜底回滚（见 transaction-mode-selection）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 表 1-2 的三态语义（1.3.2）
- **问题**: 返回值到底怎么影响命令的修改？
- **方法论的使用**: 按表 1-2 定义三态：Succeeded 保留、Failed/Cancelled 逆转。
- **结论**: 返回值 = 事务提交/回滚的遥控器。
- **结果**: Hello World 返回 Succeeded，修改被保留。

### 案例 2: 参数写入前的守卫（代码 2-26）
- **问题**: 写参数前要做什么检查？
- **方法论的使用**: `if (parameter != null && !parameter.IsReadOnly)` 检查后 `parameter.Set(value)`，失败抛异常→最终 Failed 触发回滚。
- **结论**: 失败路径主动返回/抛出，靠返回值兜底一致性。
- **结果**: 写操作失败时命令整体回滚，模型不残留半成品。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 命令批量改图元，担心"改一半失败怎么办"，想知道有没有自动回滚。
2. 用户看到 Result 枚举不知道选哪个，特别是 Cancelled 什么时候用。
3. 返回 Failed 后发现之前的修改还在（Manual 模式忘了 RollBack）——需要排查。
4. 想区分 Error 对话框和 Warning 对话框的出现条件。

### 语言信号 (用户的话里出现这些就应激活)

- "Result 返回值怎么用 / Failed 会撤销吗"
- "命令改了一半失败怎么办"
- "自动回滚 / 撤销修改"
- "Succeeded Failed Cancelled 区别"
- "result enum / auto rollback / undo command changes / return failed"

### 与相邻 skill 的区分

- 与 `revit-transaction-mode-selection` 的区别: 本 skill 是返回值语义（结果导向），transaction-mode 是前置声明（Automatic/Manual/ReadOnly）；Automatic 下两者联动。
- 与 `revit-execute-parameter-semantics` 的区别: 本 skill 讲返回值，parameter-semantics 讲三参数；返回值决定"改不改得留"，message 决定"怎么告诉用户"。
- 与 `revit-transaction-hierarchy` 的区别: transaction-hierarchy 讲手动精细控制（Transaction/SubTransaction/TransactionGroup 分层），本 skill 讲 Automatic 场景下"返回值即回滚"的免写方案。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认命令的事务模式**
   - 完成标准: 判断是 Automatic 还是 Manual。Automatic → 返回值直接驱动回滚；Manual → 需确认代码中已手动 RollBack 或依赖外部事务组。

2. **为所有失败/取消路径返回正确 Result**
   - 完成标准: 成功→`Result.Succeeded`；业务失败→`Result.Failed`（配 message 弹 Error）；用户中途取消→`Result.Cancelled`（配 message 弹 Warning）。检查没有路径漏 return 或误返回 Succeeded。
   - 判停条件: 若命令采用 Manual 事务、需要部分提交等精细控制，停——本 skill 只管返回值语义与 Automatic 回滚，精细控制指路事务三件套 skill。

3. **验证回滚行为**
   - 完成标准: 故意让命令在第 N 个图元处失败，运行后确认模型无任何残留修改（Automatic 下应自动干净回滚）。给用户说明：不要写手动撤销代码，靠返回值即可；Manual 才需要代码级控制。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- Manual 模式下的精细事务控制（部分提交、SubTransaction 回滚）——那是事务专题。
- 事件回调/Updater 内（无 Result 返回值机制）。

### 作者在书中警告的失败模式

- 所有路径都返回 Succeeded → 中途失败也被当成功，半成品修改被保留（Automatic 下不触发回滚）。
- Cancelled 误用为"返回失败"——它仍回滚，但语义是用户取消，弹 Warning。
- Manual 模式忘记 RollBack 却返回 Failed——回滚只靠外部事务组兜底，提交过的子事务可能残留。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版对 `FailureHandlingOptions` 与回滚的交互更复杂（如事务内已用 FailuresPreprocessor 处理过的警告不再回滚），书中未覆盖。
- 书未讨论大事务回滚的性能代价——自动回滚在大批量修改失败时可能很慢。

### 容易混淆的邻近方法论

- `Result.Failed`（命令级、含回滚）vs `Transaction.RollBack()`（事务级、手动）：前者是声明结果，后者是执行动作。
- "回滚"（Undo，撤销命令修改）vs "重生成"（Regenerate，更新几何）：前者是事务语义，后者是模型同步机制。

---

## 相关 skills

- revit-transaction-hierarchy：contrasts-with——本 skill 讲 Automatic 下"返回值即回滚"的免写方案，transaction-hierarchy 讲手动分层事务的精细控制，两种事务管控思路互为对照。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
