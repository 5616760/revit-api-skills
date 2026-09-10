---
name: revit-external-command-entry
description: |
  当写'被按钮触发、执行一次'的插件功能、问 Execute 签名、或 .addin FullClassName 指向的类没实现接口时调用。契约：类实现 IExternalCommand，Execute
  (commandData, ref message, elements) 返回 Result；.addin 注册 Type=Command + FullClassName。Execute 即插件的 M
  ain()。不适用：启动自动运行（Application）。trigger：外部命令、IExternalCommand、Execute 方法、external command entry。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p022
tags: [external-command, execute-contract, entry-point, csharp]
related_skills:
  - slug: revit-plugin-entry-types
    relation: composes-with
  - slug: revit-execute-parameter-semantics
    relation: composes-with
  - slug: revit-command-result-undo
    relation: composes-with
---

# 外部命令入口点：IExternalCommand 与 Execute 方法

## R — 原文 (Reading)

> 每个 Revit 插件应用程序都必须有一个进入点类来实现 IExternalCommand 接口，且必须实现 Execute() 方法。Execute() 方法是插件程序的进入点，类似于其他程序中的 Main() 方法。插件程序进入点类的定义包含在一个程序集内。
>
> — 宦国胜, 第1章 1.2.2（约 p022）

---

## I — 方法论骨架 (Interpretation)

Revit 的用户触发式插件没有 `Main()`，它靠一个**进入点契约**被 Revit 调用：

- 你的程序集里有一个类实现 `IExternalCommand`，其中实现唯一抽象方法 `Execute(ExternalCommandData commandData, ref string message, ElementSet elements)`。
- Revit 在用户点击对应按钮（或运行外部命令）时，创建这个类的实例、调用 `Execute`、然后根据返回值 `Result.Succeeded / Failed / Cancelled` 决定如何处理。
- 这个类必须在 `.addin` 清单中注册：`Type="Command"` + `FullClassName` 指向类的完整命名空间名（含程序集）。

最小契约只有两件事：**一个实现 Execute 的类** + **清单里一行注册**。缺少任何一个，Revit 都找不到你的命令。

Execute 三参数分工（详见 execute-parameter-semantics）：`commandData` 是入口上下文（拿 Application/UIDocument/视图），`message` 是错误信息输出通道，`elements` 是失败时高亮图元集合。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: Hello World 外部命令（代码 1-4）
- **问题**: 第一个插件要能被用户点击运行。
- **方法论的使用**: 新建类实现 `IExternalCommand`，在 `Execute` 中写逻辑，返回 `Result.Succeeded`；`.addin` 里 `Type="Command"` + `FullClassName` 指向该类。
- **结论**: 命令入口的最小形态 = Execute 实现 + 清单注册。
- **结果**: 命令出现在 Revit 中并可运行，成为全书的入门样板。

### 案例 2: Ribbon PushButton 绑定命令
- **问题**: 命令类的实例何时被创建？如何从按钮触发？
- **方法论的使用**: 按钮 `PushButtonData` 的 `ClassName` 指向命令类全名；Revit 点击时实例化并调用 Execute。
- **结论**: UI 控件只引用类名（字符串），不持有实例——入口由 Revit 按需创建。
- **结果**: 1.3.8 节按钮与命令的绑定流程成立。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 新建插件项目，"入口怎么写"的第一步。
2. 点击按钮没反应/报"找不到命令类"，需要检查 `FullClassName` 是否与类名一致。
3. 想确认 Execute 方法签名（参数个数、ref/out 关键字）以免编译失败。
4. 需要理解"为什么我的插件没有 Main()，代码从哪开始跑"。

### 语言信号 (用户的话里出现这些就应激活)

- "外部命令怎么写 / 入口类是什么"
- "IExternalCommand / Execute 方法"
- "按钮点了没反应 / FullClassName 找不到"
- "插件入口 / 命令入口"
- "external command entry / how to implement IExternalCommand / Execute method"

### 与相邻 skill 的区分

- 与 `revit-plugin-entry-types` 的区别: 本 skill 只深讲 Command 一种入口的最小契约；entry-types 负责三种入口的选择决策。
- 与 `revit-execute-parameter-semantics` 的区别: 本 skill 是"接口与注册"，parameter-semantics 是"Execute 的三个参数各干什么"。
- 与 `revit-command-result-undo` 的区别: 本 skill 讲入口骨架，result-undo 讲返回值如何驱动事务提交/回滚。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **创建实现 IExternalCommand 的类**
   - 完成标准: 类实现接口，`Execute(ExternalCommandData commandData, ref string message, ElementSet elements)` 签名正确（注意 `ref string` 与 `ElementSet` 类型），返回值类型为 `Result`。编译无错。

2. **在 .addin 清单中注册命令**
   - 完成标准: `<AddIn Type="Command">` 下包含正确的 `Assembly`（dll 路径）、`FullClassName`（命名空间.类名）、`AddInId`。核对类名拼写与实际一致。
   - 判停条件: 若用户是"启动时自动运行"需求 → 停，改用 IExternalApplication，并指路 `revit-plugin-entry-types`。

3. **验证加载与运行**
   - 完成标准: 重启 Revit 后命令出现在 Add-ins/功能区，点击能执行且返回预期 Result。给用户列出排查清单（Type 字段、FullClassName、程序集路径、Revit 是否信任该路径）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 插件需要启动时自动初始化、订阅事件、加 Ribbon 面板 → 用 `IExternalApplication`，不是本 skill。
- 纯后台无 UI 监听 → 用 `IExternalDBApplication`。

### 作者在书中警告的失败模式

- `Execute` 签名写错（少了 `ref`、`ElementSet` 类型用错）→ 编译失败或运行时找不到入口。
- 类名/命名空间与 `FullClassName` 不一致 → Revit 报"找不到命令"。
- 忘记处理异常：Execute 内未捕获的异常会导致 Revit 弹"未处理异常"对话框（见 execute-try-catch-message）。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版要求程序集加数字签名/受信任路径声明（`AddIn` 的 `VendorId`、程序集强名称），旧版不加签名的清单在新版可能被拒绝。
- 书未深入多命令程序集组织（一个 dll 多个命令类如何共享辅助代码），只在单命令层面讲解。

### 容易混淆的邻近方法论

- `IExternalCommand.Execute`（用户触发、一次执行）vs `IExternalApplication.OnStartup`（启动时自动、驻留）——常被搞混，判断依据是"谁先开始执行"。
- 命令类 `Execute` 里拿到的 `commandData.Application` 是 `UIApplication`，不是启动阶段的 `UIControlledApplication`——两者作用域不同。

---

## 相关 skills

- revit-plugin-entry-types：composes-with——本 skill 深讲 Command 一种入口的接口契约，entry-types 提供三入口全景选型，合起来是完整的入口知识面。
- revit-execute-parameter-semantics：composes-with——本 skill 讲接口与清单注册，parameter-semantics 讲 Execute 三参数的分工，构成命令入口的完整编程面。
- revit-command-result-undo：composes-with——本 skill 讲入口骨架，result-undo 讲返回值如何驱动事务提交/回滚，是命令流程的上下两半。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
