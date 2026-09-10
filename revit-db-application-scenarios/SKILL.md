---
name: revit-db-application-scenarios
description: |
  当需要无 UI 后台服务（文档打开自动检查、挂事件/更新器）、或分不清 IExternalDBApplication 与 IExternalApplication 时调用。DBApplication
  不添加任何 UI，用 ControlledApplication（DB 层），OnStartup/OnShutdown 返回 ExternalDBApplicationResult；只能访问数据库事件，不能建
  Ribbon/TaskDialog。不适用：任何需要 UI 的插件。trigger：后台服务、不要 UI、ControlledApplication、DB-level plugin。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p056-057
tags: [external-db-application, background-service, controlled-application, updater]
related_skills:
  - slug: revit-plugin-entry-types
    relation: depends-on
  - slug: revit-startup-shutdown-events
    relation: composes-with
---

# 数据库级外部应用程序 IExternalDBApplication 的适用场景

## R — 原文 (Reading)

> 数据库级插件是不向 Revit 用户界面添加任何内容的外部应用程序，用于为 Revit 会话分配事件和/或更新。IExternalDBApplication 接口使用 ControlledApplication 而不是 UIControlledApplication。
>
> — 宦国胜, 第1章 1.3.10 / 代码 1-37（约 p056–p057）

---

## I — 方法论骨架 (Interpretation)

Revit 的第三种入口是**数据库级外部应用程序**，专门给"不碰界面、只碰数据"的插件：

- **特征**：不向 UI 添加任何内容——没有按钮、没有面板、没有对话框。
- **用途**：在 Revit 会话中分配**事件**和**更新**——典型如文档打开时自动做合规检查、注册 Updater 动态更新模型。
- **接口差异**：`OnStartup(ControlledApplication)` / `OnShutdown(ControlledApplication)`，返回 `ExternalDBApplicationResult.Succeeded/Failed`；参数是 `ControlledApplication`（DB 层）而非 `UIControlledApplication`。
- **能力边界**：`ControlledApplication` 只提供数据库事件访问，**没有** Ribbon/TaskDialog/UI 能力。

选型判据一句话：**插件要不要长在界面上？** 要 → `IExternalApplication`；不要、只要在后台陪模型数据库 → `IExternalDBApplication`。注意它和普通 Application 一样有 OnStartup/OnShutdown 生命周期，因此事件注册/注销配对规则同样适用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 代码 1-37 数据库级应用骨架
- **问题**: 一个后台检查服务，不该打扰用户界面。
- **方法论的使用**: 实现 `IExternalDBApplication`，在 OnStartup 里用 `ControlledApplication` 订阅数据库事件。
- **结论**: DB 层入口用 DB 层上下文，天然无 UI。
- **结果**: 服务随会话加载，监听文档事件而不动界面。

### 案例 2: .addin Type="DBApplication" 注册（1.3.4）
- **问题**: 数据库级插件怎么被 Revit 识别？
- **方法论的使用**: 清单 `Type="DBApplication"`。
- **结论**: 入口类型与接口一一对应。
- **结果**: 三种入口各就各位（见 revit-plugin-entry-types）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需求是"打开文档就自动检查/自动记录"，不想弹任何窗口。
2. 要做动态模型更新（Updater）或需要进程级数据库事件。
3. 看到 IExternalDBApplication 不知道和 Application 有什么区别、什么时候用。
4. 尝试在 DBApplication 里建按钮报错，需要理解能力边界。

### 语言信号 (用户的话里出现这些就应激活)

- "后台服务 / 不要 UI / 无界面插件"
- "文档打开时自动检查合规性"
- "IExternalDBApplication / ControlledApplication"
- "数据库级外部应用程序"
- "background service / no UI plugin / DB-level application / document opened check"

### 与相邻 skill 的区分

- 与 `revit-plugin-entry-types` 的区别: 本 skill 深讲 DBApplication 一种入口；entry-types 是三入口全景选型。
- 与 `revit-startup-shutdown-events` 的区别: 本 skill 是 DB 侧应用（ControlledApplication），startup-shutdown 是 UI 侧应用（UIControlledApplication）的事件配对。
- 与批次8更新器专题（Updater）的区别: 本 skill 是入口容器，Updater 是挂在其生命周期上的具体机制（动态模型更新）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认无 UI 需求**
   - 完成标准: 明确插件不需要按钮/面板/对话框。若需要任何 UI → 停，改用 `IExternalApplication`，指路 entry-types。
   - 判停条件: 若只需要"文档打开时执行一次检查"且结果要用界面展示 → 建议"DBApplication 后台检查 + 结果写日志/共享参数"或用 Application 弹窗，二选一并说明取舍。

2. **实现接口并注册**
   - 完成标准: 类实现 `IExternalDBApplication`，OnStartup/OnShutdown 参数为 `ControlledApplication`、返回 `ExternalDBApplicationResult`；订阅的事件与 Updater 在 OnStartup 注册、OnShutdown 注销（配对）。`.addin` 的 `Type="DBApplication"`。

3. **验证能力边界与加载**
   - 完成标准: 重启 Revit 确认无 UI 加载、事件/更新器生效；确认代码中没有调用 UI 类（引用若含 RevitAPIUI.dll 需检查是否越界使用）。给用户列出 DBApplication 能做/不能做的对照清单。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 任何需要 UI 交互的插件（按钮、面板、对话框）——能力不足，直接选 Application。
- 一次性用户触发命令——那是 Command。

### 作者在书中警告的失败模式

- 在 DBApplication 中调用 UI API（TaskDialog、Ribbon）→ 运行时异常或编译依赖错误。
- 忘记 OnShutdown 注销数据库事件/Updater → 同 UI 侧一样的内存泄漏与重复触发。
- 误以为 DBApplication 可以打开文档执行完整命令流程——它只能响应事件，不能在事件里开事务（事件回调的事务约束见事件专题）。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版本（2022+）对 Updater 与 DBApplication 的注册有 `UpdaterRegistry` 持久化要求，卸载清理更复杂。
- 书未讨论"DBApplication + 共享参数"向用户暴露结果的模式，需自行扩展。

### 容易混淆的邻近方法论

- `IExternalDBApplication`（进程级入口）vs `IUpdater`（更新机制）：前者是宿主，后者是注册到宿主生命周期上的功能单元。
- `ControlledApplication`（DB 级，无 UI）vs `UIControlledApplication`（UI 级）——只差 UI 二字，但完全两个世界。

---

## 相关 skills

- revit-plugin-entry-types：depends-on——本 skill 深讲 DBApplication 一种入口的适用条件，前提是已理解三种插件入口的全景选型。
- revit-startup-shutdown-events：composes-with——DBApplication 同样具备 OnStartup/OnShutdown 生命周期，事件注册注销配对的约束规则是二者共享的。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
