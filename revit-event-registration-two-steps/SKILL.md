---
name: revit-event-registration-two-steps
description: |
  在 Revit 插件中订阅/注销事件（DocumentChanged、ViewActivated 等）时调用：何时写 handler、
  在哪注册。规范是两步：实现符合签名的事件处理函数 + 在 IExternalApplication.OnStartup 用
  ControlledApplication 尽早注册、OnShutdown 注销。不适用于：外部命令 Execute 内随手注册
  （命令对象执行后即销毁，订阅会丢失/泄漏）。Trigger 信号："订阅事件"、"register event
  handler"、"为什么事件不触发"、"OnStartup 注册"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.3.3（约p350-351）
tags: [events, registration, lifecycle, revit-api]
related_skills:
  - slug: revit-external-events-nonmodal-dialog
    relation: contrasts-with
  - slug: revit-updater-registration-triggers
    relation: composes-with
  - slug: revit-failure-definition-registration
    relation: composes-with
---

# 注册事件的两步骤框架（handler + OnStartup 注册）

## R — 原文 (Reading)

> "使用此事件是个两步骤的过程。第一步，必须有一个能处理事件通知的函数。该函数必须接受两个参数……第二步，用 Revit 注册事件。这可以通过 ControlledApplication 参数在 OnStartup()函数中尽早实现……"
>
> — 宦国胜，第5章 5.3.3（约p350-351）

---

## I — 方法论骨架 (Interpretation)

事件订阅在 Revit 插件里是一个"生命周期问题"，不只是语法问题。

- 第一步：写处理函数。签名固定——第一参数是发送者（object/Application），第二参数是事件参数类型；函数体注意只读/可写约束。
- 第二步：注册。规范位置是 `IExternalApplication.OnStartup()`，通过其中的 `ControlledApplication` 参数调用 `+=` 订阅；对应在 `OnShutdown()` 里 `-=` 注销。
- 为什么必须在 OnStartup：外部命令对象每次执行完就被销毁，里面注册的订阅随之丢失，还可能重复注册导致回调被执行多次。
- 生命周期配对原则：注册和注销必须发生在同一个持久对象的 Startup/Shutdown 里，保证"谁注册谁清理"。
- 若确实需要在命令内临时订阅（少见），必须在同一命令执行路径内配对注销，不能跨命令悬挂。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 标准事件订阅骨架

- **问题**: 插件需要在整个会话期间监听文档变化。
- **方法论的使用**: 按两步框架——先定义处理函数（双参数签名），再在 OnStartup 中经 ControlledApplication 注册；OnShutdown 注销。
- **结论**: 订阅与插件生命周期绑定，会话内稳定生效、退出时干净清理。
- **结果**: 事件可靠触发且不产生重复回调；插件卸载不残留订阅。

### 案例 2: 新手把注册写进 IExternalCommand.Execute

- **问题**: 在命令里 `app.DocumentChanged += handler`，第二次运行命令后回调被执行两次，且行为不稳定。
- **方法论的使用**: 书中生命周期规则指出命令对象每次执行后即销毁，注册应尽早放在 OnStartup；命令内注册必须同命令内注销。
- **结论**: 事件注册是应用级职责，不是命令级职责。
- **结果**: 迁移到 OnStartup/OnShutdown 配对后，重复触发与悬挂订阅消失。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 第一次给 Revit 插件加事件订阅，不知道代码写在哪。
2. 事件"有时触发有时不触发"或回调被执行多次，排查订阅位置。
3. 插件卸载/重启后出现悬挂订阅或内存泄漏。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么订阅/注册 Revit 事件"（subscribe / register Revit event）
- "OnStartup 里注册事件"（register in OnStartup）
- "事件处理程序执行了两次 / 事件不触发"（event fires twice / handler not called）

### 与相邻 skill 的区分

- 与 `revit-external-events-nonmodal-dialog` 的区别：普通事件订阅是监听 Revit 已发生的事；ExternalEvent 是请求 Revit 在空闲周期执行你的代码，注册/持有方式完全不同（Create/Raise/Dispose）。
- 与 `revit-updater-registration-triggers` 的关系：同属注册生命周期，但更新器注册需声明介入修改并配置过滤器/变更类型，本 skill 只讲普通事件的挂载配对。
- 与 `revit-failure-definition-registration` 的关系：该 skill 是故障定义与注册流程，与事件注册同为“注册类”方法论，但注册对象不同。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定义处理函数**
   - 按目标事件写出双参数签名（如 `void OnDocumentChanged(object sender, DocumentChangedEventArgs args)`）。
   - 完成标准: 签名与 API 事件委托完全匹配，函数体内明确只读/可写边界。

2. **在 OnStartup 注册、OnShutdown 注销**
   - `OnStartup(ControlledApplication a)` 中 `a.DocumentChanged += OnDocumentChanged;`；`OnShutdown` 中对应 `-=`。
   - 完成标准: 注册与注销成对出现在同一个 IExternalApplication 实现里。
   - 判停条件: 若订阅只应在某个特定会话阶段生效，可保留 OnStartup 注册但加开关标志，不要改去命令里注册。

3. **检查无重复/悬挂订阅**
   - 完成标准: 全项目搜索 `+=` 与 `-=`，数量配对；命令代码内无未注销的事件订阅。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只想"投递一次操作给 Revit 执行"——那是 ExternalEvent 的职责，普通订阅不解决线程问题。
- 命令执行期间的一次性监听——优先考虑改用事件后的轮询或重构，命令内订阅是最后手段。

### 作者在书中警告的失败模式

- 在外部命令 Execute 里注册事件 → 命令对象销毁后订阅丢失/重复注册（本单元 V2 推导）。
- 注册后从不注销 → 跨会话悬挂订阅、回调堆积。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 / .NET 4.0；新版中事件数量与签名有扩展，注册生命周期原则不变但具体事件名需查当前文档。

### 容易混淆的邻近方法论

- FailureDefinition 的注册同样应放 OnStartup（第5章 5.8.1）——"应用级一次性初始化"是同构模式。
- 更新器（IUpdater）注册也是 RegisterUpdater + OnStartup/OnShutdown 配对，见 `revit-updater-registration-triggers`。

---

## 相关 skills

- **revit-external-events-nonmodal-dialog**（External Events 框架实现非模态对话框 · contrasts-with）— 普通事件订阅是“监听 Revit 发生的事”，ExternalEvent 是“请求 Revit 做事”，方向相反。
- **revit-updater-registration-triggers**（更新器注册与触发器配置 · composes-with）— 事件注册与更新器注册同属生命周期注册，但更新器带过滤器/变更类型两个配置维度。
- **revit-failure-definition-registration**（故障定义与注册流程（FailureDefinition） · composes-with）— 事件注册生命周期与该 skill 的故障定义注册同属“注册类”方法论。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
