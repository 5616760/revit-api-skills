---
name: revit-cancelable-events-propagation
description: |
  在 Revit 前置事件（DocumentSaving、DocumentClosing、ViewExporting 等可取消事件）中要"拦截/
  取消用户操作"时调用。要点：用 Cancellable 属性判断可否取消；一旦取消，其他订阅同一事件的
  处理程序不再收到通知、对应后置事件也不触发，且取消不可逆。不适用于：DocumentChanged/
  FailuresProcessing（用 Cancel() 方法而非属性）。Trigger："取消保存"、"cancel event"、
  "阻止用户执行某操作"、"args.Cancel=true"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.3.4（约p351）
tags: [events, cancellation, pre-events, revit-api]
related_skills:
  - slug: revit-addincommandbinding-override-commands
    relation: contrasts-with
---

# 取消事件（Canceling Events）机制

## R — 原文 (Reading)

> "操作发生前即已触发的事件（如"文件保存"）往往是可取消的（使用 Cancellable 属性确定该事件是否可取消）……一旦"取消"某事件，则无法"不取消"。注：如果"前置事件"被取消，已订阅该事件的其他事件处理程序不会得到通知。"
>
> — 宦国胜，第5章 5.3.4（约p351）

---

## I — 方法论骨架 (Interpretation)

Revit 的前置事件（操作发生前触发）大多支持"一票否决"，但否决的代价和 .NET 惯例不同。

- 判断可取消性：读 `args.Cancellable` 属性，不要假设所有事件都能取消。
- 取消动作：设置 `args.Cancel = true`。注意**不可逆**——同一个事件处理流程里不能先取消再反悔。
- 传播截断语义：一个订阅者取消后，**其余订阅该事件的处理程序不再被通知**，对应的后置事件（如 DocumentSaved）也不会触发。这与 .NET 的 CancelEventArgs 只影响发起方的惯例不同。
- 工程推论：
  1. 取消判断要尽早做、尽早设 Cancel，别把副作用写在取消分支之后。
  2. 其他处理程序不能依赖"被取消事件一定会跑完所有订阅者"。
  3. 取消时应给用户反馈（提示为什么被拦截），否则用户会以为操作成功了。
- API 形态差异：DocumentChanged、FailuresProcessing 用 `Cancel()` 方法/`IsCancellable()`，不是 `Cancel` 属性——写代码前先确认形态。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 拦截不合规的保存操作

- **问题**: 项目要求不满足校验条件时阻止用户保存文档。
- **方法论的使用**: 订阅 DocumentSaving（前置事件），检查 Cancellable 后在校验失败分支设 `args.Cancel = true`，并提示用户原因。
- **结论**: 保存被拦截，且取消不可逆；订阅方顺序在后校验逻辑也应独立成立。
- **结果**: 用户看到明确拦截原因，文件未被写入。

### 案例 2: 多个处理程序订阅同一前置事件

- **问题**: 三个处理程序都订阅 DocumentSaving，第一个设了 Cancel=true，后两个的行为。
- **方法论的使用**: 按传播截断语义推导——后两个不再被通知，DocumentSaved 也不触发。
- **结论**: 取消动作影响全局传播，多订阅者设计必须假设"自己可能收不到通知"。
- **结果**: 设计上把关键校验放在最早执行的订阅者中，并保证各订阅者相互独立。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要"阻止用户保存/关闭/导出/打印不合规文档"这类合规拦截。
2. 多个插件订阅同一事件时排查"我的处理程序没被调用"。
3. 在事件参数上写 `Cancel` 却发现编译不过或行为怪异。

### 语言信号 (用户的话里出现这些就应激活)

- "取消/阻止 保存、关闭、导出"（cancel / prevent saving, closing, exporting）
- "args.Cancel = true 没生效 / 后面的 handler 不执行"
- "Cancellable 属性"（Cancellable property）

### 与相邻 skill 的区分

- 与 `revit-addincommandbinding-override-commands` 的区别：命令绑定的 CanExecute 灰化是“事前禁止入口”；本 skill 的取消事件是“操作发起后一票否决”，拦截时机与手段不同。
- 与 `revit-documentclosing-no-model-edit` 的区别：该 skill 讲只读事件里不能改模型；本 skill 讲如何正确取消事件本身及取消后的传播后果。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认事件类型与取消形态**
   - 查 API：该事件是否前置可取消（`Cancellable` 属性）；若是 Cancel()/IsCancellable() 形态（DocumentChanged/FailuresProcessing）按方法调用写。
   - 完成标准: 明确记录"属性还是方法"及可取消性结论。

2. **实现取消逻辑**
   - 校验失败 → 设 Cancel（尽早）；向用户输出拦截原因；取消分支之后的代码假设其他订阅者可能已被跳过。
   - 完成标准: 取消路径有用户反馈；无"先取消再恢复"的非法尝试。
   - 判停条件: 若事件不可取消（Cancellable == false），改为事后补偿（DocumentChanged 提示、IUpdater 修正），到此停止。

3. **验证多订阅者独立性**
   - 完成标准: 任意订阅者被取消截断后，其余逻辑不依赖该事件继续传播。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 后置事件（DocumentChanged、DocumentSaved 等已发生事件）——取消没有意义，本就不支持属性取消。
- 想做"条件性禁用命令入口"——用 AddInCommandBinding 的 CanExecute 更合适。

### 作者在书中警告的失败模式

- 以为取消只影响自己 → 实际截断所有后续订阅者与后置事件。
- 忽略 FailuresProcessing 用 Cancel() 方法而非 Cancel 属性 → 写错 API 形态。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版新增/调整了部分可取消事件，且"取消后是否通知后续订阅者"在新版个别事件上有差异，以当前文档为准。

### 容易混淆的邻近方法论

- .NET CancelEventArgs 惯例：语义不同，勿凭 WinForms/WPF 经验推断。
- 与后置事件只读通知（DocumentChanged）构成前置/后置事件分类框架。

---

## 相关 skills

- **revit-addincommandbinding-override-commands**（重写 Revit 命令（AddInCommandBinding）框架 · contrasts-with）— 事件取消是操作发起后一票否决，命令绑定 CanExecute 灰化是事前禁止入口。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
