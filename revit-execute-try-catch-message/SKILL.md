---
name: revit-execute-try-catch-message
description: |
  当插件命令弹'未处理的异常'、或要把命令从调试版改造成发布版时调用。Execute 必须 try-catch-finally：catch 中 message=ex.Message 并返回 Result
  .Failed，Revit 弹友好错误对话框；finally 释放资源。不要在 Execute 裸跑业务。不适用：调试期故意冒泡。trigger：未处理的异常、message=ex.Message、u
  nhandled exception、try catch in command。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p025-026
tags: [error-handling, try-catch, external-command, robustness]
related_skills:
  - slug: revit-execute-parameter-semantics
    relation: depends-on
  - slug: revit-external-command-entry
    relation: composes-with
  - slug: revit-command-result-undo
    relation: composes-with
  - slug: revit-failures-preprocessor
    relation: composes-with
---

# Execute 入口必须 try-catch 并将异常 message 返回

## R — 原文 (Reading)

> 这可用于辅助调试运行的命令。部署给用户的命令应该在该示例入口方法中运用"try.. catch.. finally"，以免 Revit 捕获异常。代码 1-4 在 catch 中将 message = ex.Message 并返回 Result.Failed。
>
> — 宦国胜, 第1章 1.2.2 故障排除 / 代码 1-4（约 p025–p026）

---

## I — 方法论骨架 (Interpretation)

Revit 的命令运行在你的代码里，但你控制不了 Revit 怎么处理你抛出的异常。如果不捕获，异常会冒泡到 Revit 层面，表现为"未处理的异常"对话框——用户看到一堆堆栈，无法理解，插件也显得不可靠。

正确模式是三层闭环：

1. **try**：把业务逻辑全部放进 try 块。
2. **catch**：捕获异常，把可读信息写进 Execute 的 `message` 参数（`message = ex.Message`），返回 `Result.Failed`。Revit 会用 `message` 弹出一个友好错误对话框。
3. **finally**：释放非托管资源（如临时文件、COM 引用），保证清理必然执行。

要点：`message` 不是日志，是"给用户看的一句话错误说明"——所以不要塞整段堆栈，取 `ex.Message` 或更友好的一句话即可。这是 Revit 特有的命令级错误反馈协议：**错误信息通过返回值 + message 参数回传，而不是靠异常冒泡**。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 代码 1-4 的 Hello World 入口
- **问题**: 用户运行命令时若内部出错，怎么给友好反馈而非崩掉？
- **方法论的使用**: Execute 中 try-catch-finally 包裹，catch 里 `message = ex.Message` 并返回 `Result.Failed`。
- **结论**: 异常信息通过 message 通道回传给 Revit 对话框。
- **结果**: 命令失败时用户看到具体错误，而不是"未处理的异常"。

### 案例 2: 故障排除清单（s1-x01）
- **问题**: 常见失败原因排查时，"Execute 内异常未被捕获"是高频问题。
- **方法论的使用**: 将未捕获异常列为故障清单首查项，用 try-catch 兜底修复。
- **结论**: 入口异常处理是命令健壮性的第一道防线。
- **结果**: 第 4 章故障处理专题把异常处理贯穿所有 Execute 实践。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件点一下，Revit 弹"未处理的异常"，用户想知道为什么并修复。
2. 要把实验性命令改造成发布版，需要补齐错误处理。
3. 用户想知道 message 参数到底拿来干嘛、Result.Failed 和 message 的关系。
4. 命令里有多处可能抛异常（参数为空、图元被删、单位不合法），需要统一兜底。

### 语言信号 (用户的话里出现这些就应激活)

- "未处理的异常 / unhandled exception"
- "命令报错怎么弹给用户"
- "Execute 里要 try catch 吗 / message 参数怎么用"
- "插件崩溃 / 一点就报错"
- "message = ex.Message / return Result.Failed"

### 与相邻 skill 的区分

- 与 `revit-execute-parameter-semantics` 的区别: 本 skill 是"异常如何进 message 并返回 Failed"，parameter-semantics 是"三参数的完整分工"（含 elements 高亮）。
- 与 `revit-command-result-undo` 的区别: 本 skill 关注错误传递，result-undo 关注 Failed 返回后 Revit 如何自动回滚模型。
- 与 `revit-failures-preprocessor` 的区别: 本 skill 是命令入口兜底，failures-preprocessor 是事务内 Revit 失败机制（如删除冲突）的自定义处理。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **检查 Execute 是否已有 try-catch**
   - 完成标准: 定位 Execute 方法，确认业务代码是否包裹在 try 中。若无 → 按步骤 2 重构；若有 → 检查 catch 是否做了 message 回传。

2. **重构为 try-catch-finally 结构**
   - 完成标准: catch 块设置 `message = ex.Message`（或提炼一句可读信息）并 `return Result.Failed`；finally 块释放非托管资源。确保所有 return 路径都有 Result 值（含 catch 外可能的兜底 return）。
   - 判停条件: 若用户是调试期（想打印堆栈到输出窗口），可在 catch 里加 `TaskDialog`/`Debug.WriteLine` 输出完整堆栈，再 return Failed；发布版不得向用户暴露堆栈。

3. **验证错误路径**
   - 完成标准: 构造一个必然失败的输入（如空选集、删除已删图元），确认 Revit 弹出含 message 内容的错误对话框而非"未处理异常"。给用户说明 message 的字数/语气建议（一句话、面向用户）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 事务内部的失败处理（Revit 的 FailureDefinition/IFailuresPreprocessor 机制）——那是批次2故障处理专题。
- 事件回调（如 DocumentChanged）内不能简单照搬 return Failed——事件没有 Result 概念。

### 作者在书中警告的失败模式

- catch 后不设置 message 直接 return Failed → 用户看到空对话框或无内容错误。
- catch 中继续抛异常或 return 错误类型不匹配 → 未捕获异常再次冒泡。
- 把全部业务塞进 try 但 finally 里做可能抛异常的操作 → 掩盖原始错误。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：`message` 对话框样式在新版本仍适用，但 `FailureHandlingOptions`/对话框自定义（2021+）提供了更细的错误交互，书中未涉及。
- 书未强调异常的性能成本——高循环内 try-catch 影响性能，应在入口兜底而非每元素 catch。

### 容易混淆的邻近方法论

- `message` 参数（用户可读错误）vs 日志文件（开发者可查细节）：一个是面向用户，一个是面向开发。
- `Result.Failed`（命令级失败）vs `throw`（异常）：前者走 Revit 协议通道，后者会打断 Revit 控制流——命令里应优先前者。

---

## 相关 skills

- revit-execute-parameter-semantics：depends-on——本 skill 用 message 参数回传错误，前提是先理解 Execute 三参数的分工语义。
- revit-external-command-entry：composes-with——本 skill 是命令入口骨架的健壮性增强，与 entry 的接口契约共同构成完整入口形态。
- revit-command-result-undo：composes-with——本 skill 把异常转成 message + Result.Failed，result-undo 讲 Failed 返回后如何自动回滚模型。
- revit-failures-preprocessor：composes-with——本 skill 是命令入口的异常兜底，failures-preprocessor 是事务内失败机制的自定义处理，两层防御互补。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
