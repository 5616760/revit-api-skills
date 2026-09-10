---
name: revit-shared-parameter-file-init
description: |
  当需通过 API 访问共享参数文件且面临路径初始化或文件切换时调用。
  不适用于：已通过 UI 绑定共享参数、可扩展存储。
  关键 trigger 信号："OpenSharedParameterFile 抛异常"、"共享参数文件路径 shared parameter file path"、"未初始化 not initialized"、"一次只持有一个文件 one file at a time"。
  核心：必须先设 SharedParametersFilename 再调 OpenSharedParameterFile，Revit 不记住上次路径。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.2.2 (约p318)
tags: [shared-parameter-file, initialization, open-shared-parameter-file, revit-api]
related_skills: []
---

# 共享参数文件路径必须由 OpenSharedParameterFile 初始化，否则会抛异常

## R — 原文 (Reading)

> 在使用之前，初始化该属性。若未初始化则引发异常。默认情况下，Revit 无共享参数文件。在 Revit Application Options 对象中设置共享参数文件的文件名称。若文件不存在则引发异常。
>
> — 宦国胜, 第5章 5.1.2.2 (约p318)

---

## I — 方法论骨架 (Interpretation)

Revit 通过 API 访问共享参数文件有三个关键约束：

1. **必须先设路径再载入**：`app.Options.SharedParametersFilename = filePath` → `app.OpenSharedParameterFile()`，顺序不能颠倒。若未设路径就调 OpenSharedParameterFile，抛异常。
2. **文件必须存在**：路径指向的文件必须真实存在于磁盘，否则抛异常。默认情况下 Revit 无共享参数文件。
3. **一次只持有一个文件**：Revit 不维护"上次成功路径"，每次载入新文件即替换当前。插件每次启动必须显式指定路径。

这意味着插件使用自己的共享参数时，正确的加载顺序是：设插件自带路径 → OpenSharedParameterFile 载入 → 操作 → 切回用户文件路径 → 再 OpenSharedParameterFile。这是"沙箱式"用法，避免污染用户全局参数定义。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 插件不污染用户共享参数文件

- **问题**: 插件要在不污染用户共享参数定义的前提下使用自己的参数
- **方法论的使用**: 遵循"设路径→载入→操作→切回"的沙箱式顺序——先记录用户当前路径，切换到插件自带路径并 OpenSharedParameterFile，操作完后切回用户文件
- **结论**: 插件参数定义不污染用户全局参数定义，用户无感知
- **结果**: 插件参数文件独立管理，用户原有参数文件不受影响

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件启动时需要载入自带的共享参数文件
2. OpenSharedParameterFile 抛异常，需要诊断路径是否已设、文件是否存在
3. 插件需要在用户共享参数文件和自带文件间切换
4. 多个插件共享参数文件冲突

### 语言信号 (用户的话里出现这些就应激活)

- "OpenSharedParameterFile 抛异常 / open shared parameter file exception"
- "共享参数文件路径 / shared parameter file path"
- "未初始化 / not initialized / 没有共享参数文件"
- "切换参数文件 / switch parameter file / 一次只持有一个文件"

### 与相邻 skill 的区分

- 与 `revit-shared-parameter-file-swap` 的区别：该 skill 是完整的三步切换法案例；本 skill 是路径初始化的底层约束与异常防御。
- 与 `revit-data-storage-paths` 的区别：该 skill 是共享参数 vs 可扩展存储选型；本 skill 是共享参数文件的路径管理。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **设置共享参数文件路径**
   - `app.Options.SharedParametersFilename = filePath;`
   - 确认文件真实存在于磁盘
   - 完成标准: 路径已设且文件存在

2. **载入共享参数文件**
   - `DefinitionFile sharedParamsFile = app.OpenSharedParameterFile();`
   - 完成标准: 返回非 null 的 DefinitionFile 对象
   - 判停条件: 若抛异常，检查路径是否已设、文件是否存在，修复后重试

3. **操作完成后切回原始路径（若需要）**
   - 记录操作前的用户路径 → 操作 → 切回用户路径 → 再 OpenSharedParameterFile
   - 完成标准: 用户参数文件恢复

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 可扩展存储——不走共享参数文件路径
- 已通过 UI 绑定共享参数且无需 API 管理文件路径

### 作者在书中警告的失败模式

- 未设路径就调 OpenSharedParameterFile → 抛异常
- 路径指向不存在的文件 → 抛异常
- 忘记切回用户文件 → 污染用户全局参数定义

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，"一次只持有一个文件"的约束可能在新版本中放宽

### 容易混淆的邻近方法论

- SharedParametersFilename（设路径属性）vs OpenSharedParameterFile（载入方法）——前者设路径，后者实际打开文件
- 共享参数文件路径 vs 共享参数 GUID——前者管文件级访问，后者管定义级唯一标识

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
