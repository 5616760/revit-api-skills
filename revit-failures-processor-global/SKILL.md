---
name: revit-failures-processor-global
description: |
  让插件接管 Revit 全部错误对话框时使用。规则：会话中只有一个活动 IFailuresProcessor；注册后标准错误对话框不再出现；Dismiss(Document) 清理挂起 UI；WaitForUserInput 挂起事务。何时调用：企业级统一错误 UI。何时不调用：只想安静批处理。Trigger：'接管错误对话框/替代标准 UI/WaitForUserInput/Dismiss'（global failures processor, replace error dialog）。高风险：WaitForUserInput 无 UI → 事务无限挂起。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.2（约p374-375）
tags: [failures-processor, global, error-ui, wait-for-input, takeover]
related_skills:
  - slug: revit-failures-processing-steps
    relation: depends-on
---

# IFailuresProcessor 全局故障处理器

## R — 原文 (Reading)

> Revit 会话中只能有一个活动的 IFailuresProcessor……如果 Revit 插件选择为 Revit 注册一个故障处理器，则该处理器将成为所有 Revit 会话错误的默认错误处理程序，而标准的 Revit 错误对话则不会出现。
>
> — 宦国胜, 第5章 5.8.2 故障处理器（约p374-375）

---

## I — 方法论骨架 (Interpretation)

`IFailuresProcessor` 是故障流水线的**最后一棒**，也是**权力最大、责任最重**的一棒：

- **唯一性**：整个 Revit 会话只能有一个活动的全局处理器。
- **接管 UI**：注册之后，标准的 Revit 错误对话框**从此不再出现**——所有会话错误（不只是你插件的）都走你的处理器。这是"接管整个错误 UI"级别的责任。
- **时机**：在 FailuresProcessing 事件处理**之后**获得最终控制，是兜底。
- **两个特殊方法**：
  - `Dismiss(Document)`：清理**挂起的 UI**——比如弹窗还留在屏幕上，调用它把它关掉/清理。
  - `WaitForUserInput`：处理器可以返回"等待用户输入"状态，把事务**挂起**，直到用户操作（提交/回滚）完成。前提是**处理器自己要把 UI 保留在屏幕上**——如果返回 WaitForUserInput 但屏幕上没有任何保留的 UI，事务将**无限挂起**，文件相当于被冻结。

方法论要点：全局处理器是"最后防线"，风险与责任并存。能用事务级预处理器（revit-failures-preprocessor）或事件（revit-failures-processing-event）解决的，别轻易上全局处理器。一旦上了，就必须把 Dismiss 与 WaitForUserInput 的生命周期管好，否则要么无 UI 冻结、要么残留 UI 无法清理。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: WaitForUserInput 却没有 UI
- **问题**: V2 预测场景——处理器返回 WaitForUserInput 但屏幕上没有保留任何 UI，会怎样？
- **方法论的使用**: 检查契约——WaitForUserInput 要求处理器自己保留 UI 在屏幕上。
- **结论**: 事务无限挂起，文件相当于被冻结。
- **结果**: 用户操作不了、事务结束不了——必须保证返回 WaitForUserInput 前 UI 已显示。

### 案例 2: 注册全局处理器接管错误 UI
- **问题**: 插件要实现企业级统一错误 UI。
- **方法论的使用**: 注册 IFailuresProcessor——注册后标准错误对话框不再出现。
- **结论**: 全局处理器成为所有会话错误的默认处理程序。
- **结果**: 全统一错误体验，但责任巨大（所有错误都归你管）。

### 案例 3: Dismiss 清理挂起 UI
- **问题**: 弹窗留在屏幕上，需要程序化清理。
- **方法论的使用**: Dismiss(Document) 清理挂起的 UI。
- **结论**: 结束与处理器相关的 UI 生命周期。
- **结果**: 无残留弹窗干扰后续操作。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要做"企业级统一错误界面"，不希望用户看到零散的 Revit 原生对话框。
2. 需要所有错误都经过自己的逻辑（记录、上报、按规则处理）。
3. 排查"事务挂起/文件冻结"——怀疑 WaitForUserInput 无 UI 或 Dismiss 未清理。
4. 决定"要不要上全局处理器"的架构决策。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么接管 Revit 的错误对话框？" / "replace the standard Revit error dialog"
- "全局故障处理器 IFailuresProcessor" / "global failures processor"
- "事务挂起不动了 / 文件冻结" / "transaction hangs, file frozen"
- "WaitForUserInput / Dismiss 怎么用？" / "how to use WaitForUserInput and Dismiss"
- "所有错误都走我的界面" / "route all errors to my UI"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-processing-steps` 的关系：本 skill 是三步总览框架的第三棒（兜底）。与 `revit-failures-preprocessor`、`revit-failures-processing-event` 的边界：前两者分别是事务级与事件级处理，不接管 UI；本 skill 是唯一接管全部错误 UI 的全会话级处理器——能不用全局就别用。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **评估是否真的需要全局处理器**：
   - 问："这些故障能靠事务级预处理器或事件解决吗？"
   - 能 → 不上全局处理器（保持默认 UI）。
   - 必须统一 UI/兜底 → 继续步骤 2。
   - 完成标准: 决策有明确理由；知道这是"接管全部错误 UI"级别的责任。

2. **实现并注册**：
   - 实现 `IFailuresProcessor`：处理逻辑 + Dismiss(Document) 清理挂起 UI + 正确的 FailureProcessingResult 返回。
   - 按 Revit 要求注册（应用启动时），确保全局唯一。
   - 完成标准: 注册成功；处理逻辑收敛（不会无限循环）。

3. **管理 UI 生命周期**：
   - 若返回 WaitForUserInput：**先显示/保留处理器自己的 UI 在屏幕上**，再返回该结果。
   - 用 Dismiss(Document) 清理挂起 UI，避免残留弹窗。
   - 完成标准: 无 UI 冻结、无残留弹窗；事务最终收敛到明确状态。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只想安静批处理——事件/预处理器足够，全局处理器大材小用且责任过重。
- 不想管理 UI 生命周期——别碰全局处理器。
- 插件是工具型小插件——接管全局错误 UI 可能破坏用户对 Revit 原生交互的预期。

### 作者在书中警告的失败模式

- **WaitForUserInput 无 UI → 事务无限挂起**——文件冻结，最严重的坑。
- **注册后标准错误对话框不再出现**——所有错误都归你管，漏处理就是用户无感知。
- **Dismiss 不清 → 挂起 UI 残留**——污染后续操作。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版中 IFailuresProcessor 的注册方式与 Dismiss 行为细节有调整，应按当前 SDK 核对。
- 未讨论"多个插件都想注册全局处理器"的冲突——只能有一个，谁先注册谁赢，多插件环境要小心。

### 容易混淆的邻近方法论

- `IFailuresProcessor`（全局、接管 UI、最后） vs `IFailuresPreprocessor`（事务级、不接管 UI、最先）：名字只差 Pre，责任差一个量级。
- WaitForUserInput（挂起等用户，前提有 UI） vs ProceedWithCommit/RollBack（立即收敛）：一个是"等"，两个是"走"。

---

## 相关 skills

- revit-failures-processing-steps（depends-on）：本 skill 是三步流水线的最后一棒（兜底），总览见该 skill。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
