---
name: revit-command-app-loading-timing
description: |
  当需要'启动时自动执行/订阅事件'却只写了外部命令、或分不清命令与应用谁先加载时调用。IExternalCommand 空闲时由用户点击触发、执行完即销毁；IExternalApplication 启
  动时自动调 OnStartup、关闭时 OnShutdown、全程驻留。启动初始化/事件订阅/建 Ribbon 必须用 Application。不适用：命令内业务逻辑。trigger：启动时自动运行、
  OnStartup、load at startup。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p026-031
tags: [plugin-architecture, lifecycle, external-application, external-command]
related_skills:
  - slug: revit-plugin-entry-types
    relation: composes-with
  - slug: revit-startup-shutdown-events
    relation: composes-with
---

# 外部命令与外部应用程序的加载时机差异

## R — 原文 (Reading)

> 添加面板项目不同于 Hello World 项目，因为在 Revit 运行时它是自动调用的。此项目使用 IExternalApplication 接口。当 Revit 启动时会调用外部应用程序，而 Revit 关闭时会自动卸载。
>
> — 宦国胜, 第1章 1.2.3 / 1.3.1（约 p026–p031）

---

## I — 方法论骨架 (Interpretation)

两种入口的本质差异是**"什么时候被唤醒、活多久"**：

- **外部命令（IExternalCommand）**：Revit 空闲时、由用户点击按钮才被实例化并调用 `Execute`；执行完返回值即销毁。它是"被动的一次性调用"。
- **外部应用程序（IExternalApplication）**：Revit 启动时自动调用 `OnStartup`，关闭时调用 `OnShutdown`；在 Revit 整个生命周期内驻留。它是"主动的驻留程序"。

由此推导两条铁律：

1. **启动时要做的事（订阅文档事件、建 Ribbon 面板、初始化缓存）只能放 OnStartup**——Command 根本没有"启动时刻"。
2. **用户触发的一次性操作只能放 Command**——Application 不会替你弹个按钮给用户点。

两者是搭档关系：Application 负责"起床后的准备"，Command 负责"用户来时的服务"。一个成熟插件通常是 Application 注册按钮 + Command 执行动作 + Application 挂事件。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 添加面板项目（1.2.3）
- **问题**: 插件要随 Revit 启动自动出现在功能区。
- **方法论的使用**: 改用 `IExternalApplication`，在 `OnStartup` 中创建 Ribbon 面板。
- **结论**: UI 定制只能在启动时刻进行。
- **结果**: 面板随 Revit 启动自动加载。

### 案例 2: 命令启用条件（1.3.2）
- **问题**: 什么时候才能启用外部命令？
- **方法论的使用**: Revit 在无活动命令/编辑模式（空闲）时启用已注册命令。
- **结论**: 命令是被动等待用户，不可自动运行。
- **结果**: 命令按钮在空闲时可用、执行后释放控制权。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需求是"打开 Revit 就自动弹个窗口 / 自动检查模型"，但已经用 Command 实现了却发现不触发。
2. 想知道 OnStartup / OnShutdown 何时被调用、是不是每次开文档都调。
3. 设计混合插件："启动时订阅事件 + 按钮触发操作"，要理清职责划分。
4. 插件卸载不干净（面板残留、事件没退订）怀疑是入口类型选错。

### 语言信号 (用户的话里出现这些就应激活)

- "启动时自动运行 / 自动加载"
- "OnStartup / OnShutdown 什么时候调用"
- "外部命令和外部应用程序有什么区别"
- "我想让插件打开 Revit 就执行"
- "run at startup / load on startup / application vs command lifecycle"

### 与相邻 skill 的区分

- 与 `revit-plugin-entry-types` 的区别: 本 skill 讲两种入口的**时序行为差异**，entry-types 讲**选型 + .addin 字段**；两者互补，选型后可来这里理解行为。
- 与 `revit-startup-shutdown-events` 的区别: 本 skill 是"命令 vs 应用"的横向对比，startup-shutdown 深讲 Application 内部 OnStartup/OnShutdown 里事件注册/注销的配对约束。
- 与 `revit-db-application-scenarios` 的区别: 本 skill 对比的是 UI 侧两种入口；DBApplication 是第三类（无 UI）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定需求归属哪类时刻**
   - 完成标准: 写出需求清单并逐条打标：属于"启动时/常驻"还是"用户点击时"。若含"启动时"条目 → 必须用 Application（OnStartup）；若纯用户触发 → Command。

2. **检查现有实现是否入口错配**
   - 完成标准: 若现在用 Command 却想启动运行 → 提示改为 Application 并指路 entry-types；若混合需求 → 给出 Application+Command 双类方案（Application 注册按钮与事件，Command 执行动作）。
   - 判停条件: 若用户同时需要"无 UI 后台监听"，停并指路 `revit-db-application-scenarios`——那不是本 skill 范畴。

3. **验证时序正确性**
   - 完成标准: 用输出窗口/日志验证：Application 在 Revit 启动时调用 OnStartup、关闭时 OnShutdown；Command 在点击时才 Execute。给用户总结"谁在什么时刻活着"的对照表。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只要写一个命令内部逻辑（Execute 里干什么）——那与加载时机无关。
- 无 UI 后台服务——那是 DBApplication 范畴。

### 作者在书中警告的失败模式

- 在 Command 里做"启动初始化"——永远不触发，因为命令没有启动时刻。
- 在 Application 里放"用户操作逻辑"——没有入口让用户调用它。
- 忘记 OnShutdown 清理（退订事件/释放资源）——插件关闭后残留（见 startup-shutdown-events）。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：较新版本引入 `AppDomain` 隔离与 `AddInAvailability` 选择器，加载时机控制更细，书中未覆盖。
- 书未讨论"零文档启动"（Revit 打开无项目时 OnStartup 的行为）——实际开发中需处理活动文档为 null 的情况。

### 容易混淆的邻近方法论

- "Revit 启动时调用 Application" ≠ "每个文档打开都调用"——`OnStartup` 只在进程启动时一次，文档级事件（DocumentOpened）是另一回事。
- `UIApplication`（命令执行时上下文）vs `UIControlledApplication`（OnStartup 参数）：前者只能被正在运行命令的程序持有，后者是启动阶段专用。

---

## 相关 skills

- revit-plugin-entry-types：composes-with——本 skill 讲两种入口的时序行为差异，entry-types 讲选型与 .addin 字段，合起来是入口知识整体。
- revit-startup-shutdown-events：composes-with——本 skill 是"命令 vs 应用"的横向对比，startup-shutdown 深讲 Application 内部事件注册注销配对，构成纵向深化。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
