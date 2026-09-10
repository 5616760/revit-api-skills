---
name: revit-external-events-nonmodal-dialog
description: |
  非模态对话框/停靠面板的按钮要执行 Revit API 操作时调用。核心：非模态窗体不在 Revit 主线程，
  直接调 API 抛异常，必须走四步：实现 IExternalEventHandler.Execute → ExternalEvent.Create() →
  按钮点击只调 Raise() → Revit 空闲周期回调 Execute；窗体关闭时 Dispose。不适用于模态对话框
  （ShowDialog 期间可直接调 API）。Trigger："modeless/非模态"、"跨线程调 API 报错"、"空闲事件"、
  "面板按钮调 API"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.4（约p352-355）
tags: [external-events, modeless-dialog, threading, revit-api]
related_skills:
  - slug: revit-iupdater-execute-transaction-rules
    relation: contrasts-with
---

# External Events 框架实现非模态对话框

## R — 原文 (Reading)

> "Revit API 提供了一个 External Events 框架，以适应使用非模态对话框。要使用 External Events 框架来实现非模态对话框，遵循下列步骤：(1)通过从 IExternalEventHandler 接口派生……(2)用静态方法 ExternalEvent.Create()……"
>
> — 宦国胜，第5章 5.4（约p352-355）

---

## I — 方法论骨架 (Interpretation)

ExternalEvent 是 Revit 提供的"线程投递"机制：把非主线程的意图转成主线程的合法执行。

- 问题根源：非模态对话框不阻塞 Revit 主线程，窗体按钮运行在自己的 UI 线程上；Revit API 只允许在主线程（API 上下文）调用，线程外调 API 抛异常。
- 四步框架：
  1. 写 handler：类实现 `IExternalEventHandler`，业务逻辑（读模型/改图元，含事务）写在 `Execute()` 里。
  2. 创建：`ExternalEvent.Create(handler)` 得到 ExternalEvent 实例，交给非模态窗体持有。
  3. 触发：窗体按钮点击只调 `externalEvent.Raise()`——立即返回，不阻塞 UI。
  4. 执行：Revit 在下一个空闲周期回调 `Execute()`，此时处于合法 API 上下文。
- 生命周期：释放责任在窗体侧——`OnFormClosed`/Closing 中 Dispose 掉 ExternalEvent 与 handler。
- 心智模型：Raise 是"发消息进队列"，Execute 是"主线程消费消息"；类似空闲事件（idle event）的默认频率调度。
- 状态传递：Execute 拿不到按钮上下文，需在 handler 里放字段（如 enum 标记本次要做什么）供 Raise 前设置。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 非模态面板上的"刷新统计"按钮

- **问题**: 面板按钮点击后要读取模型统计数据，直接在按钮事件里调 Revit API 抛异常。
- **方法论的使用**: 按四步框架——统计读取逻辑写进 IExternalEventHandler.Execute()；Create 后由面板持有；按钮点击调 Raise()；Revit 空闲周期执行读取；面板关闭时 Dispose。
- **结论**: UI 线程只负责 Raise，所有 API 调用收敛到 Execute。
- **结果**: 非模态面板持续可用，按钮触发后统计数据在空闲周期被安全读取并回显。

### 案例 2: 窗体生命周期与释放

- **问题**: 关闭非模态对话框后偶发崩溃或泄漏。
- **方法论的使用**: 书中要求对话框持有 handler 并在 OnFormClosed 中 Dispose ExternalEvent 与 handler。
- **结论**: ExternalEvent 的创建与销毁必须与窗体生命周期绑定。
- **结果**: 关闭面板后不再有悬挂的 handler 被 Revit 回调。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 做 WPF/WinForms 非模态工具面板（或停靠面板），按钮要读/改模型。
2. 后台线程/定时器要触发 Revit 操作（如监控外部数据变化后更新图元）。
3. 非模态窗体里调 API 抛 "invalid thread"/"外部异常" 类错误。

### 语言信号 (用户的话里出现这些就应激活)

- "非模态对话框 / modeless dialog / 停靠面板 dockable panel"
- "ExternalEvent / IExternalEventHandler / Raise()"
- "空闲事件 idle event"、"跨线程调 Revit API 报错"（cross-thread API call）

### 与相邻 skill 的区分

- 与 `revit-iupdater-execute-transaction-rules` 的区别：IUpdater 由“模型变更”触发且修改并入原事务；ExternalEvent 由“用户/线程意图”触发且自己开事务。
- 与 `revit-event-registration-two-steps` 的区别：普通事件订阅是“监听 Revit 发生的事”；ExternalEvent 是“请求 Revit 替你做事”，方向相反且不经过 OnStartup 注册。
- 与模态对话框的区别：模态场景可直接调 API；非模态必须走 Raise。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **实现 handler**
   - 类实现 `IExternalEventHandler`，`Execute(UIApplication app)` 内写全部 API 逻辑（含事务、只读检查）。
   - 完成标准: handler 无 UI 依赖；需要区分的动作通过 handler 字段/枚举传递。

2. **创建并交给窗体持有**
   - 在窗体构造/显示前 `ExternalEvent.Create(handler)`，实例存窗体字段。
   - 完成标准: ExternalEvent 生命周期完全跟随窗体实例。

3. **按钮只调 Raise，关闭时 Dispose**
   - 完成标准: 所有按钮回调内只有 `Raise()`（及状态字段赋值）；`OnFormClosed` 中 Dispose ExternalEvent。
   - 判停条件: 若发现窗体实际是 ShowDialog 模态调用，可退回直接调 API 方案，但需确认模态期间无重入需求。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 模态对话框（ShowDialog）：模态期间可直接调 API，无需投递。
- 一次性批量处理任务：用 IExternalCommand 按钮即可，ExternalEvent 是为常驻非模态 UI 服务的。

### 作者在书中警告的失败模式

- 线程外直接调 API / 开事务 → 异常（第5章 5.2 线程约束）。
- 窗体关闭不 Dispose → 悬挂 handler、后续回调崩溃。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，无 DockablePane API 详解（新版停靠面板是该模式的主要载体，但 ExternalEvent 原理不变）；WPF Dispatcher.Invoke 的常规跨线程方案在 Revit 中不适用，勿混用。

### 容易混淆的邻近方法论

- Idling 事件：External Events "操作类似于默认频率空闲事件"，但更轻量、专为此场景设计。
- ExternalCommand: 命令有自己的 API 上下文，无需投递。

---

## 相关 skills

- **revit-iupdater-execute-transaction-rules**（IUpdater Execute 方法约束与事务规则 · contrasts-with）— IUpdater 由模型变更触发且改模型并入原事务，ExternalEvent 由用户/线程意图触发且自己开事务。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
