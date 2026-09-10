---
name: revit-addincommandbinding-override-commands
description: |
  要重写/拦截/禁用 Revit 内置命令时调用：CreateAddInCommandBinding() + RevitCommandId 绑定，
  订阅 BeforeExecuted / CanExecute / Executed 三事件（CanExecute 返回 false 即灰化命令）。命令
  ID 用 LookupCommandId("ID_XXX") 查找，可从 Revit 日志 Jrn.Command 条目提取；先查
  CanHaveBinding。程序触发命令请用 PostCommand。Trigger："禁用/重写内置命令"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.7（约p363-365）
tags: [commands, command-binding, ui-extension, revit-api]
related_skills:
  - slug: revit-postcommand-single-limit
    relation: contrasts-with
---

# 重写 Revit 命令（AddInCommandBinding）框架

## R — 原文 (Reading)

> "AddInCommandBinding 类可用于重写现有 Revit 命令。它有三个替换现有命令实现有关事件：BeforeExecuted……CanExecute……Executed……要创建命令绑定，调用 UIApplication.CreateAddInCommandBinding()……"
>
> — 宦国胜，第5章 5.7（约p363-365）

---

## I — 方法论骨架 (Interpretation)

AddInCommandBinding 让插件"接管"内置命令的行为，是命令级的扩展点。

- 三个事件构成完整的命令生命周期钩子：
  - **BeforeExecuted**：命令执行前触发（可做前置准备/记录）。
  - **CanExecute**：返回 false 时命令被禁用（按钮灰化）——上下文条件禁用的标准手段。
  - **Executed**：命令执行后触发（可做后处理）。
- 创建流程：
  1. 找到命令 ID：`RevitCommandId.LookupCommandId("ID_XXX")` 按 ID 字串查，或 `LookupPostableCommandId(PostableCommand 枚举)`。
  2. 找不到 ID 字串时，从 Revit 日志的 `Jrn.Command` 条目里挖（用户手动操作一次，日志里就有）。
  3. 确认可绑定：`commandId.CanHaveBinding`——不是所有命令都能被重写。
  4. `UIApplication.CreateAddInCommandBinding(commandId)` 创建绑定并订阅事件。
- 生命周期：绑定在 OnStartup 建立，OnShutdown 中移除。
- 典型用途：特定上下文禁止命令（CanExecute 返回 false）、给内置命令加前后钩子。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 禁止在特定上下文执行"编辑设计选项"

- **问题**: 插件要在特定模型状态下阻止用户执行"编辑设计选项"内置命令。
- **方法论的使用**: 从 Revit 日志的 Jrn.Command 条目提取命令 ID（如 "ID_EDIT_DESIGNOPTIONS"），LookupCommandId 取得 RevitCommandId，CanHaveBinding 确认后 CreateAddInCommandBinding，订阅 CanExecute 在禁用上下文返回 false。
- **结论**: 命令入口被灰化，无需轮询拦截。
- **结果**: 用户在不允许的状态下无法启动该命令，其他时候命令正常。

### 案例 2: 给内置命令加执行后钩子

- **问题**: 想在用户每次执行某内置命令后自动做检查。
- **方法论的使用**: 绑定后订阅 Executed 事件，回调中执行检查逻辑（遵守事件只读约束）。
- **结论**: 三个事件覆盖命令前/可否/命令后三个切点。
- **结果**: 检查随命令自动执行，用户无感知接入。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要禁用/灰化某些内置命令（防止用户在受控模型里乱改）。
2. 要在内置命令前后注入逻辑（校验、日志、联动）。
3. 不知道目标内置命令的 ID 字串怎么获取。

### 语言信号 (用户的话里出现这些就应激活)

- "禁用/重写/拦截 内置命令"（disable / override built-in command）
- "AddInCommandBinding / CanExecute / BeforeExecuted"
- "LookupCommandId / RevitCommandId / 命令 ID 从日志里找"（find command id from journal）

### 与相邻 skill 的区分

- 与 `revit-postcommand-single-limit` 的区别：本 skill 是向内拦截/接管 Revit 命令（AddInCommandBinding）；该 skill 是向外发起命令（PostCommand），两者共用 RevitCommandId 体系但方向相反。
- 与 `revit-cancelable-events-propagation` 的区别：事件取消是操作发起后否决；CanExecute 灰化是事前禁用入口，体验更干净。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位命令 ID**
   - 优先 LookupPostableCommandId(枚举)；否则从日志 Jrn.Command 提取 ID 字串后 LookupCommandId。
   - 完成标准: 拿到非空 RevitCommandId，记录其来源。

2. **确认可绑定并创建绑定**
   - `commandId.CanHaveBinding == true` 才继续；OnStartup 中 CreateAddInCommandBinding 并按需求订阅三事件；OnShutdown 移除。
   - 完成标准: 绑定创建成功且生命周期成对。
   - 判停条件: 若 CanHaveBinding 为 false，告知用户该命令不可重写，改用事件拦截或 UI 移除方案，到此停止。

3. **实现目标行为并验证**
   - 禁用场景：CanExecute 按上下文返回 false；钩子场景：BeforeExecuted/Executed 写逻辑。
   - 完成标准: 目标上下文中命令灰化（或钩子按预期触发），非目标上下文行为不变。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只想"替用户执行"某命令 → 用 PostCommand，不要绑死命令。
- 要拦截的是文档级操作（保存/关闭）→ 用可取消事件，不必动命令。

### 作者在书中警告的失败模式

- 未检查 CanHaveBinding 直接绑定 → 部分命令绑定失败或行为未定义。
- OnShutdown 不移除绑定 → 会话状态残留。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；部分核心命令（如保存相关）在新版中不可绑定或行为有调整；日志格式在不同版本间有差异。

### 容易混淆的邻近方法论

- IExternalCommandAvailability（第1章）：控制的是**你自己的外部命令**何时可用；CommandBinding 控制的是**Revit 内置命令**——对象不同。

---

## 相关 skills

- **revit-postcommand-single-limit**（发布命令（PostCommand）的限制与检查 · contrasts-with）— CommandBinding 是向内拦截/接管命令，PostCommand 是向外发起命令，两者共用 RevitCommandId。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
