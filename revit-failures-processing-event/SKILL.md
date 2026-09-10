---
name: revit-failures-processing-event
description: |
  做无 UI 自动故障处理时使用。FailuresProcessing 事件在预处理器后触发，可有任意数量处理程序；事件无返回值，必须用 args.SetProcessingResult() 传回状态。何时调用：全局自动去警告/按标准自动解决。何时不调用：只影响单事务。Trigger：'自动处理故障无弹窗/SetProcessingResult'（failures processing event, automatic failure handling）。陷阱：ProceedWithCommit 无解决方案会无限循环。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.2（约p373-374）
tags: [failures-processing, event, auto-handling, set-processing-result, no-ui]
related_skills:
  - slug: revit-failures-processing-steps
    relation: depends-on
  - slug: revit-failures-processor-global
    relation: contrasts-with
---

# FailuresProcessing 事件机制

## R — 原文 (Reading)

> FailuresProcessing 事件最适合那些想提供无用户界面的自定义故障处理的应用程序……FailuresProcessing 事件会在 IFailuresPreprocessor（如果有的话）完成后引发。……
>
> — 宦国胜, 第5章 5.8.2 故障处理事件（约p373-374）

---

## I — 方法论骨架 (Interpretation)

`FailuresProcessing` 事件是故障流水线的**第二棒**，定位很明确：**给"无 UI 的自动故障处理"提供全局入口**。

几个关键事实：

- **触发时机**：在事务级预处理器（如果有）**之后**触发——预处理器没搞定的事，轮到事件处理器。
- **数量**：可以挂**任意数量**的处理程序，没有唯一性限制。
- **通信方式**：.NET 事件处理器签名没有返回值，所以**状态必须通过事件参数 `SetProcessingResult(FailureProcessingResult)` 写回**——这是唯一的回传通道，别试图 return。
- **适用场景**：自动去警告（DeleteWarning）、按办公标准自动解决（如"自动把重叠墙删掉"）、无弹窗批处理。

两个必须记住的陷阱：
1. **ProceedWithCommit 但没真正解决问题 → 无限循环**：故障引擎发现故障还在，会再触发一轮 FailuresProcessing。返回 Commit 前必须确认解决方案真实生效。
2. **事务已回滚，返回 Commit → 被当 RollBack**：结果码必须与事务实际状态一致。

它与预处理器/全局处理器的定位差异：预处理器是"事务级、最先、唯一"；事件是"全局、居中、多个"；全局处理器是"最后、替代标准 UI"。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 事件处理器如何传回状态
- **问题**: V2 预测场景——事件处理程序没有返回值，怎么把"我已处理完这些故障"的状态传回故障引擎？
- **方法论的使用**: 用事件参数 args.SetProcessingResult(FailureProcessingResult) 传递。
- **结论**: 事件签名无返回值，这是唯一通道。
- **结果**: 状态成功传回，流水线按预期继续。

### 案例 2: 无 UI 自动故障处理
- **问题**: 插件要全自动处理故障，不想有任何对话框。
- **方法论的使用**: 订阅 FailuresProcessing 事件，在处理器里 DeleteWarning/自动解决，SetProcessingResult 收尾。
- **结论**: 该机制专为无 UI 场景设计。
- **结果**: 批处理全程静默自动收敛。

### 案例 3: 无限循环陷阱
- **问题**: 返回 ProceedWithCommit 却没有成功解决方案会怎样？
- **方法论的使用**: 认识到引擎会再次触发处理循环。
- **结论**: 会导致无限循环。
- **结果**: 设计时强制"Commit 必有真实解决动作"。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要做"全自动、无弹窗"的故障处理。
2. 需要按办公标准自动解决某些故障（不留给用户决策）。
3. 事件处理器里不知道怎么把结果传回引擎。
4. 排查"处理完又反复处理 / 事务卡死"——怀疑无限循环。

### 语言信号 (用户的话里出现这些就应激活)

- "无 UI 自动处理故障" / "automatic failure handling without UI"
- "FailuresProcessing 事件怎么订阅？" / "subscribe to FailuresProcessing event"
- "事件里怎么返回处理结果？" / "how to return result from an event handler"
- "SetProcessingResult 是干嘛的？" / "what does SetProcessingResult do"
- "处理完警告又重复出现/死循环" / "infinite loop in failure processing"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-processor-global` 区分：事件在全局处理器之前触发、不接管 UI，适合无 UI 自动处理；全局处理器替代标准 UI 兜底。与 `revit-failures-processing-steps` 的关系：本 skill 是三步总览框架里的第二棒，触发时机与结果回传详见总览。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **订阅事件**：在应用启动处订阅 `Application.FailuresProcessing` 事件，事件处理器签名 `(object sender, FailuresProcessingEventArgs args)`。
   - 完成标准: 订阅成功；确定该处理器职责（自动去警告 / 自动解决 / 记录日志）。

2. **处理故障并写回结果**：
   - 用 `args.GetFailuresAccessor()` 读故障、处置（DeleteWarning / ResolveFailure 等）。
   - 必须 `args.SetProcessingResult(result)` 传回状态。
   - 若返回 ProceedWithCommit，先确认故障已真实解决；若事务已回滚，别请求 Commit。
   - 完成标准: 每个分支都有明确的结果码，且与事实一致。

3. **验证收敛**：
   - 构造带故障的事务测试：处理链一轮或数轮内收敛（最终 Commit 或 RollBack）。
   - 完成标准: 无无限循环、无静默失败、无异常冒泡。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只影响单个事务的压噪——用事务级预处理器更精准（revit-failures-preprocessor）。
- 需要完全接管错误 UI——用全局 IFailuresProcessor（revit-failures-processor-global）。
- 用户要保留弹窗决策——不要订阅事件自动处理。

### 作者在书中警告的失败模式

- **事件处理程序无法返回"值"**——忘了 SetProcessingResult 等于没处理，引擎按未处理继续。
- **ProceedWithCommit 无解决方案 → 无限循环**。
- **事务已回滚后请求 Commit → 被当 RollBack**。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 FailuresProcessingEventArgs 的 API 形态与额外能力（如访问更多上下文）有演进，应按当前 SDK 核对。
- 未讨论"多个事件处理程序之间的执行顺序与冲突"——若多个处理器互相矛盾，结果不可预测，工程上要谨慎叠加。

### 容易混淆的邻近方法论

- FailuresProcessing 事件 vs DocumentChanged 等文档事件：前者是故障引擎的回调（可影响提交/回滚裁决）；后者是只读通知（不可改模型）。
- `SetProcessingResult` vs 直接 return：事件处理器必须用前者，后者无效。

---

## 相关 skills

- revit-failures-processing-steps（depends-on）：本 skill 是三步流水线的第二棒，总览见该 skill。
- revit-failures-processor-global（contrasts-with）：事件在全局处理器之前且不接管 UI；全局处理器替代标准 UI 兜底。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
