---
name: revit-shared-parameter-file-swap
description: |
  当插件需要载入自带共享参数文件而不影响用户原有参数文件时调用三步切换法。
  不适用于：用户手动管理参数文件、可扩展存储。
  关键 trigger 信号："插件自带参数文件 plugin own parameter file"、"不污染用户参数 don't affect user parameters"、"切换-载入-切回 swap-load-restore"、"GUID 跨会话唯一 GUID unique across sessions"。
  核心流程：记录用户路径 → 切换到插件文件并载入 → 操作完成切回用户文件。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录B FAQ B.1 (约p426)
tags: [shared-parameter, file-swap, plugin-isolation, guid, revit-api]
related_skills:
  - slug: revit-shared-parameter-file-init
    relation: depends-on
  - slug: revit-binding-type-vs-instance
    relation: composes-with
  - slug: revit-shared-parameter-guid-check
    relation: composes-with
---

# 程序加载共享参数文件而不影响用户

## R — 原文 (Reading)

> Revit 可以使用多个共享参数文件，但一次只能从一个文件中读取参数……API 应用程序应避免影响用户参数文件。应用程序应附有其自身的参数文件来包含要用的参数。要将参数载入 Revit 文件，则：① 应用程序必须知道用户参数文件名；② 切换到应用程序参数文件并载入参数；③ 然后切换回用户文件。
>
> — 宦国胜, 附录B FAQ B.1 (约p426)

---

## I — 方法论骨架 (Interpretation)

Revit 共享参数管理的特有机制：一次只能持有一个共享参数文件，但可以通过"切换-载入-切回"三步法实现插件参数与用户参数的隔离。

三步切换法：
1. **记录**：记录用户当前参数文件路径（`app.Options.SharedParametersFilename`）
2. **切换+载入**：切换到插件自带的参数文件路径 → `app.OpenSharedParameterFile()` 载入 → 在此文件中操作参数定义
3. **切回**：操作完成后，切回用户原始参数文件路径 → 再 `OpenSharedParameterFile()` 恢复

原理：Revit 不维护"上次成功路径"，每次载入新文件即替换当前。共享参数的 GUID 确保跨会话和模型唯一性——即使参数文件切换，已绑定到图元的参数定义因 GUID 而稳定。这样插件参数定义不污染用户全局参数定义，用户无感知。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 插件不干扰用户共享参数文件

- **问题**: 插件需要在 Revit 文件中添加自定义共享参数，但用户的共享参数文件正在被其他插件使用
- **方法论的使用**: 三步切换法——① 记录用户当前参数文件路径；② 切换到插件自带参数文件路径并 OpenSharedParameterFile 载入参数定义到 Revit 文件；③ 切回用户原始参数文件路径
- **结论**: 插件参数定义不污染用户全局参数定义，用户无感知
- **结果**: 插件参数成功载入，用户参数文件恢复原状

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件启动时需要载入自带的共享参数定义而不干扰用户
2. 多个插件共享参数文件冲突
3. 需要理解共享参数 GUID 的跨会话唯一性
4. 插件卸载时需要清理参数文件引用

### 语言信号 (用户的话里出现这些就应激活)

- "插件自带参数文件 / plugin own parameter file"
- "不污染用户参数 / don't affect user parameters / 不影响用户"
- "切换-载入-切回 / swap-load-restore / 三步切换法"
- "GUID 跨会话唯一 / GUID unique across sessions"

### 与相邻 skill 的区分

- 与 `revit-shared-parameter-file-init` 的关系：本 skill 是完整的三步切换法案例实践，依赖该 skill 的路径初始化与异常防御。
- 与 `revit-shared-parameter-guid-check` 的关系：本 skill 关注参数文件隔离；该 skill 关注复制后 GUID 的定义级唯一性，切换后需配套检查。
- 与 `revit-binding-type-vs-instance` 的关系：切换文件后重新绑定参数时复用该 skill 的绑定模式选型。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **记录用户当前参数文件路径**
   - `string userFilePath = app.Options.SharedParametersFilename;`
   - 完成标准: 成功记录用户路径

2. **切换到插件参数文件并载入参数定义**
   - `app.Options.SharedParametersFilename = pluginFilePath;`
   - `DefinitionFile pluginFile = app.OpenSharedParameterFile();`
   - 在此文件中操作参数定义（添加/获取 Definition）
   - 将参数定义绑定到 Revit 文件中的图元类别
   - 完成标准: 参数定义成功载入 Revit 文件
   - 判停条件: 若 OpenSharedParameterFile 抛异常，检查插件文件路径是否正确、文件是否存在

3. **切回用户原始参数文件路径**
   - `app.Options.SharedParametersFilename = userFilePath;`
   - `app.OpenSharedParameterFile();`
   - 完成标准: 用户参数文件恢复

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 可扩展存储——不走共享参数文件
- 用户手动管理参数文件——无需代码切换
- 无需隔离的简单场景——直接用用户文件即可

### 作者在书中警告的失败模式

- 忘记切回用户文件 → 污染用户全局参数定义
- Revit 不记住上次路径 → 每次启动必须显式指定

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，"一次只持有一个文件"的约束可能在新版本中放宽

### 容易混淆的邻近方法论

- 共享参数文件切换（文件级隔离）vs 共享参数 GUID（定义级唯一标识）——前者管文件访问，后者管定义唯一性
- 三步切换法 vs 直接用用户文件——前者隔离，后者共享

---

## 相关 skills

- **revit-shared-parameter-file-init**（共享参数文件路径必须由 OpenSharedParameterFile 初始化，否则会抛异常 · depends-on）— 本 skill 的三步切换法依赖该 skill 的路径初始化约束。
- **revit-binding-type-vs-instance**（共享参数绑定（Binding）的两种模式：TypeBinding vs InstanceBinding · composes-with）— 切换前后重绑定时复用该 skill 的绑定模式知识。
- **revit-shared-parameter-guid-check**（共享参数复制需检查 GUID · composes-with）— 复制切换后需按该 skill 检查 GUID 定义级唯一性。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
