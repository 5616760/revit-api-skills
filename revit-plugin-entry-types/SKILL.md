---
name: revit-plugin-entry-types
description: |
  当新建 Revit 插件或 .addin 加载失败、纠结选 IExternalCommand / IExternalApplication / IExternalDBApplication 时调用。按
  生命周期判断：点击一次执行→Command；启动初始化+事件+UI→Application；无 UI 后台监听→DBApplication。不适用于：已在写具体业务逻辑。trigger：addin T
  ype 字段、插件入口、外部命令和应用程序区别、entry point / addin manifest。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p038-039
tags: [plugin-architecture, addin-manifest, entry-point, revit-api]
related_skills: []
---

# 三种插件入口与对应 .addin Type 字段决策

## R — 原文 (Reading)

> 代码 1-17/1-18/1-19：.addin 清单中 AddIn Type 字段分别填 "Command"、"Application"、"DBApplication"，与外部命令、外部应用程序、数据库级外部应用程序三种插件入口一一对应。
>
> — 宦国胜, 第1章 1.3.4（约 p038–p039）

---

## I — 方法论骨架 (Interpretation)

Revit 插件不是"写个类就能跑"，而是通过 `.addin` 清单文件把入口类注册给 Revit。清单里最关键的是 `Type` 字段，它声明了这个插件属于哪种入口，而每种入口对应一种生命周期：

- `Type="Command"` → 类实现 `IExternalCommand`：用户点按钮时才执行一次，执行完对象销毁。
- `Type="Application"` → 类实现 `IExternalApplication`：Revit 启动时自动调用 `OnStartup`，关闭时调用 `OnShutdown`，用于初始化和驻留。
- `Type="DBApplication"` → 类实现 `IExternalDBApplication`：无 UI 的数据库级入口，专门挂事件和后台更新。

决策本质是回答"我的插件需要在什么时刻活着"：被用户叫醒一次（Command）、陪 Revit 全程运行（Application）、还是只陪模型数据库运行（DBApplication）。`Type` 字段拼错或与接口不匹配，Revit 会直接拒绝加载该插件。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: Hello World 外部命令
- **问题**: 第一个插件只要"被点击时弹一句话"，最小入口是什么？
- **方法论的使用**: 用 `IExternalCommand` 实现 `Execute()`，清单写 `Type="Command"`。
- **结论**: 命令入口满足"用户触发、一次执行"的全部需求。
- **结果**: 代码 1-17 对应的 Hello World 在 Revit 中可被外部命令运行。

### 案例 2: Ribbon 面板应用
- **问题**: 插件要在功能区加自己的面板，且需要随 Revit 启停。
- **方法论的使用**: 用 `IExternalApplication`，在 `OnStartup` 里创建 Ribbon 面板，清单写 `Type="Application"`。
- **结论**: 只有 Application 入口能在启动时拿到 UIControlledApplication 定制界面。
- **结果**: 代码 1-18 对应项目在 Revit 启动时自动加载面板。

### 案例 3: 数据库级应用
- **问题**: 一个纯后台服务，不碰 UI，但要监听文档事件。
- **方法论的使用**: 用 `IExternalDBApplication` + `ControlledApplication`，清单写 `Type="DBApplication"`。
- **结论**: DBApplication 对应 DB 层，不依赖 RevitAPIUI.dll。
- **结果**: 代码 1-19 对应项目在 Revit 会话中分配事件与更新。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 新建插件项目，第一行就问"该继承哪个接口 / 入口类怎么写"。
2. `.addin` 清单加载失败、Revit 不识别插件，怀疑 `Type` 字段或 `FullClassName` 写错。
3. 需要一个"启动时自动执行/挂后台监听"的功能，但刚学了外部命令却实现不了。
4. 代码评审时看到一个"又做 UI 又做后台"的入口，需要判断是否设计错了。

### 语言信号 (用户的话里出现这些就应激活)

- "addin 文件怎么写 / Type 字段填什么"
- "外部命令、外部应用程序、数据库级应用有什么区别"
- "我要做一个插件，入口是什么"
- "启动时自动运行 / 后台监听 / 不要 UI"
- "entry point / addin manifest / IExternalDBApplication vs IExternalApplication"

### 与相邻 skill 的区分

- 与 `revit-external-command-entry` 的区别: 本 skill 是"三种入口的横向决策"，command-entry 只深入 Command 一种的接口契约。
- 与 `revit-command-app-loading-timing` 的区别: 本 skill 聚焦"选哪种入口+清单字段"，loading-timing 聚焦"选完之后两者何时被调用"的行为差异。
- 与 `revit-db-application-scenarios` 的区别: 本 skill 覆盖三入口全景，db-application-scenarios 只讲 DBApplication 的适用条件。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确插件生命周期需求**
   - 完成标准: 回答出"插件是否需要在启动/关闭时自动运行"以及"是否需要 UI 或挂事件"。若答案含糊，追问用户是"点一下执行"还是"常驻监听"。
   - 判停条件: 若用户能明确说出"点按钮执行一次"→ 跳到步骤 2 选 Command；若说"启动时初始化 + 有 UI"→ 选 Application；若说"无 UI 后台监听"→ 选 DBApplication。

2. **选择入口接口并写出类骨架**
   - 完成标准: 类实现对应接口：`IExternalCommand`(Execute) / `IExternalApplication`(OnStartup/OnShutdown) / `IExternalDBApplication`(OnStartup/OnShutdown, 返回 ExternalDBApplicationResult)。确认选用的参数类型与返回类型与该入口匹配。

3. **填写 .addin 清单并验证**
   - 完成标准: `.addin` 的 `Type` 字段为 Command/Application/DBApplication 之一且与接口严格对应，`Assembly`/`FullClassName`/`AddInId` 正确。检查后提示用户重启 Revit 用"Add-ins"面板验证加载，并告知常见失败点（Type 大小写、类名拼写、程序集路径）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已经在写某个入口的具体业务逻辑（参数读取、图元创建）——那是对应专题 skill 的范畴。
- 插件只有普通工具类、没有 Revit 入口需求（如纯数据转换库）。

### 作者在书中警告的失败模式

- `.addin` 的 `Type` 与接口不匹配或路径错误时，Revit 静默/报错拒绝加载——清单是插件能否被识别的第一道闸门。
- 误以为"外部命令可以像 Main() 一样随意调用其他插件"——每个命令类必须独立注册。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0：较新版本要求 `.addin` 的 VendorId、VendorDescription、以及 Revit 2022+ 的部分入口行为有调整（如 `Journaling`、受信任路径）。老清单在新版本可能加载失败。
- 书未覆盖 `EntryPoint` 之外的 `Updater`/`AddInType` 组合应用（DBApplication 与 Updater 协同的场景只有 DB 事件部分提及）。

### 容易混淆的邻近方法论

- "外部命令"与"外部应用程序"名称近似，易混——前者是被动触发一次执行，后者是主动驻留管理生命周期。
- `Type="Application"` 与 `Type="DBApplication"` 只差 `DB` 前缀，但一个绑定 RevitAPIUI（Ribbon/TaskDialog），一个只能用 RevitAPI（ControlledApplication）。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
