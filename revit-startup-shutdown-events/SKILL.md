---
name: revit-startup-shutdown-events
description: |
  当写 IExternalApplication、要 OnStartup 订阅事件/OnShutdown 注销、或插件卸载后事件残留/内存泄漏时调用。铁律：事件注册注销成对——OnStartup 中 +
  =、OnShutdown 中 -=；漏注销致内存泄漏或关闭异常。UIControlledApplication 提供事件访问与 Ribbon 定制。不适用：Command 入口。trigger：OnS
  tartup、事件没注销、内存泄漏、subscribe on startup unsubscribe on shutdown。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p036-037
tags: [external-application, events, lifecycle, memory-leak]
related_skills: []
---

# 外部应用程序：在 OnStartup 注册事件、在 OnShutdown 注销事件

## R — 原文 (Reading)

> 外部应用程序接口有两个抽象方法，即 OnStartup() 和 OnShutdown()。Revit 在启动时调用 OnStartup()，关闭时调用 OnShutdown()。代码 1-16 在 OnStartup 注册、OnShutdown 注销 DialogBoxShowing 事件。
>
> — 宦国胜, 第1章 1.3.3（约 p036–p037）

---

## I — 方法论骨架 (Interpretation)

`IExternalApplication` 的两个钩子构成插件的**进程级生命周期**：

- `OnStartup(UIControlledApplication)`：Revit 启动时调用。这里做一切"开机准备"——订阅需要监听的事件（如 `DocumentOpened`、`DialogBoxShowing`）、创建 Ribbon 面板/选项卡、初始化静态缓存。
- `OnShutdown(UIControlledApplication)`：Revit 关闭时调用。这里做一切"关机清理"——`-=` 注销 OnStartup 里注册的每个事件、释放资源。

核心约束是**对称性**：OnStartup 里每注册一个事件，OnShutdown 里必须注销一个。破坏对称的后果：

- 漏注销 → 事件处理函数仍被引用 → 内存泄漏（重复加载插件时尤其明显）；
- 更糟时 Revit 关闭过程中事件回调异常，导致关闭卡住或崩溃。

`UIControlledApplication` 是启动/关闭阶段专用上下文：它给事件访问、Ribbon 定制能力，但它**不是** `UIApplication`（没有活动文档的模型访问能力）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 代码 1-16 的 DialogBoxShowing 订阅
- **问题**: 想在 Revit 弹对话框时自动处理，事件从哪订阅？
- **方法论的使用**: `OnStartup` 里 `app.DialogBoxShowing += handler`，`OnShutdown` 里 `-=` 注销。
- **结论**: 事件注册/注销成对出现在两个生命周期钩子中。
- **结果**: 事件在插件整个生命周期内有效且干净退出。

### 案例 2: Ribbon 面板在 OnStartup 创建（1.2.3）
- **问题**: 面板什么时候建？
- **方法论的使用**: 在 OnStartup 中用 UIControlledApplication 创建 Ribbon 面板/选项卡。
- **结论**: UI 定制与事件订阅同属"启动准备"。
- **结果**: 面板随启动加载（见 revit-command-app-loading-timing）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要监听文档打开/关闭/保存事件，不知道在哪注册。
2. 卸载或关闭 Revit 时报错、或反复加载插件后内存暴涨，怀疑事件没退订。
3. 写 OnStartup/OnShutdown 时不确定参数类型（UIControlledApplication 还是别的）。
4. 需要"启动时初始化 + 关闭时清理"的完整生命周期样板。

### 语言信号 (用户的话里出现这些就应激活)

- "启动时订阅事件 / 关闭时注销"
- "OnStartup / OnShutdown 怎么写"
- "插件卸载后事件还在 / 内存泄漏"
- "Revit 关闭时报错 / 退订事件"
- "subscribe events on startup / unsubscribe on shutdown / memory leak / UIControlledApplication"

### 与相邻 skill 的区分

- 与 `revit-command-app-loading-timing` 的区别: 本 skill 深讲 Application 内部两个钩子的对称约束；loading-timing 讲 Command 与 Application 的横向时序差异。
- 与 `revit-db-application-scenarios` 的区别: 本 skill 是 UI 侧应用（UIControlledApplication + Ribbon），DB 侧应用用 ControlledApplication 且无 UI。
- 与批次8事件专题的区别: 本 skill 是"事件在生命周期钩子里如何注册/注销"，事件专题是"各个事件的语义与数据"。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **盘点需要订阅的事件清单**
   - 完成标准: 列出插件要监听的全部事件（如 DocumentOpened、DialogBoxShowing），每个都标注"OnStartup 注册"。确认这些事件在 UIControlledApplication/ControlledApplication 上可用。

2. **成对编写注册与注销代码**
   - 完成标准: OnStartup 中每个事件一个 `+=`，OnShutdown 中对应一个 `-=`（同一 handler 引用）；静态缓存与资源同样成对初始化/清理。逐个核对没有"只注册不注销"的事件。
   - 判停条件: 若用户的目标是"无 UI 后台监听"，停并指路 `revit-db-application-scenarios`——本 skill 的 UIControlledApplication 不适用。

3. **验证对称性与退出行为**
   - 完成标准: 日志确认启动时全部注册、关闭时全部注销；反复加载/卸载插件数次后无内存增长；Revit 正常关闭。给用户一句铁律：注册与注销必须在两个钩子中成对出现。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- Command 入口（无生命周期钩子，别在 Execute 里订阅常驻事件）。
- 非模态对话框/后台线程里的事件访问——线程亲和限制属于事件/外部事件专题。

### 作者在书中警告的失败模式

- OnShutdown 漏注销 → 内存泄漏、二次加载时事件重复触发（一次修改执行多次回调）。
- OnShutdown 里访问已销毁对象（如试图操作已关闭文档）→ 关闭时异常。
- OnStartup 抛异常 → 插件加载失败，Revit 可能报插件错误。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版对 OnShutdown 期间可做操作的限制更严（部分事件在关闭期不可访问），需额外防护。
- 书未强调"一次加载失败后再次加载"的状态残留问题，实践中常需静态标志位防重复注册。

### 容易混淆的邻近方法论

- `OnStartup`/`OnShutdown`（进程级，一次）vs `DocumentOpened` 等文档级事件（每次文档）——前者是生命周期的骨架，后者是挂在骨架上的业务。
- `UIControlledApplication`（启动期）vs `UIApplication`（命令期）——名字相近，能力与生命周期完全不同。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
