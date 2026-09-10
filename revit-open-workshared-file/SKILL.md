---
name: revit-open-workshared-file
description: |
  要以编程方式打开中心文件/工作共享文件并控制加载范围时调用：先用 WorksharingUtils.
  GetUserWorksetInfo(modelPath) 在打开前拿到用户工作集信息 → 筛选目标 WorksetId → 构造
  WorksetConfiguration（CloseAll 后只 Open 目标工作集）→ OpenDocumentFile(modelPath,
  openOptions)。系统工作集自动打开，用户工作集按需。不适用于：单机文件直接 Open。Trigger：
  "打开中心文件"、"只加载部分工作集"、"WorksetConfiguration"、"最小化内存"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.13.3 打开工作共享文件（约p399-400）
tags: [worksharing, workset-configuration, document-open, revit-api]
related_skills:
  - slug: revit-worksharing-api-decision-framework
    relation: composes-with
  - slug: revit-transmissiondata-offline
    relation: contrasts-with
---

# 打开工作共享文件决策流程

## R — 原文 (Reading)

> "Application.OpenDocumentFile (ModelPath, OpenOptions) 方法可用于设置打开工作共享文件相关的选项。除了从中心文件分离或允许本地文件由其所有者以外的用户以只读方式打开选项外，还可以设置相关的工作集选项。"
>
> — 宦国胜，第5章 5.13.3 打开工作共享文件 约p399–p400

---

## I — 方法论骨架 (Interpretation)

打开工作共享文件不是"一个 Open 调用"，而是"先侦察、再配置、后打开"的三段流程。

- 侦察：打开前用 `WorksharingUtils.GetUserWorksetInfo(modelPath)` 获取文件的全部用户工作集信息——不打开文档就能拿到，是控制加载范围的前提。
- 筛选：从工作集信息里筛出要加载的目标 `WorksetId` 列表。
- 配置：构造 `WorksetConfiguration`：
  - `CloseAll()` 关闭所有用户工作集，再 `Open(worksetIds)` 只打开目标集合；
  - 记住语义：**系统工作集（如标高、轴网视图相关）会自动打开**，用户工作集才受配置控制。
- 打开：`Application.OpenDocumentFile(modelPath, openOptions)`，OpenOptions 里还可设置：
  - 从中心文件**分离**（Detach）；
  - 允许本地文件被非所有者以**只读**方式打开。
- 价值：只加载需要的工作集，避免大中心文件全量载入内存——200MB 中心文件只看一个工作集时尤其关键。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 最小内存打开 200MB 中心文件

- **问题**: 只想查看特定工作集的图元，直接 OpenDocumentFile 会全量加载。
- **方法论的使用**: 先 GetUserWorksetInfo 拿到工作集清单，构造 WorksetConfiguration 执行 CloseAll() 后仅 Open 目标 WorksetId，再 OpenDocumentFile。
- **结论**: 用户工作集按需加载 + 系统工作集自动打开 = 内存占用最小。
- **结果**: 大文件打开内存与耗时显著下降，未加载工作集的图元不进入内存。

### 案例 2: 审阅用途的分离/只读打开

- **问题**: 需要以非所有者身份查看他人本地文件，或打开中心文件副本做审阅。
- **方法论的使用**: OpenOptions 设置只读打开或从中心分离选项。
- **结论**: 打开选项覆盖"分离/只读/工作集"三类控制。
- **结果**: 审阅场景不产生所有权冲突，分离副本可独立修改。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要打开中心文件做批量检查/统计，不想全量加载。
2. 后台服务/批处理要打开工作共享文件，内存受限。
3. 要以只读或分离方式打开他人文件避免冲突。

### 语言信号 (用户的话里出现这些就应激活)

- "打开中心文件 / open central file programmatically"
- "WorksetConfiguration / GetUserWorksetInfo"
- "只加载部分工作集 / 最小化内存"（load only some worksets / minimize memory）

### 与相邻 skill 的区分

本 skill 与 `revit-worksharing-api-decision-framework` 配合：那个管"打开之后"的所有权操作（检出/放弃），本 skill 管"打开之时"的加载范围与打开方式——前后衔接。与 `revit-transmissiondata-offline` 区分：两者都是"打开前预读文件信息"路径，但对象不同——本 skill 读工作集，那个读外部链接。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **打开前侦察工作集**
   - `WorksharingUtils.GetUserWorksetInfo(modelPath)` 列出全部用户工作集。
   - 完成标准: 拿到工作集清单（含 Id 与名称），尚未打开文档。

2. **构造 WorksetConfiguration**
   - `CloseAll()` 后对目标集合逐个/批量 `Open(worksetId)`；明确记录"系统工作集自动打开"的预期。
   - 完成标准: 配置中只包含确定要加载的用户工作集。
   - 判停条件: 需要全部工作集 → 可不配置（默认），但仍建议显式表达意图。

3. **带选项打开**
   - OpenOptions 装入 WorksetConfiguration（及分离/只读需求），OpenDocumentFile(modelPath, openOptions)。
   - 完成标准: 打开后抽查非目标工作集图元未加载（如 collector 数量），目标工作集可正常访问。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单机（非共享）文件——无工作集概念，直接 Open。
- 需要编辑全模型的场景——过度收窄加载范围会导致后续检出/修改失败，不如全量打开。

### 作者在书中警告的失败模式

- 假设系统工作集也受配置控制 → 它们永远自动打开，配置对它们无效。
- 直接 OpenDocumentFile 全量加载大中心文件 → 内存/时间浪费。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版 OpenOptions 与链接工作集配置有扩展（如按需加载链接），流程骨架不变。

### 容易混淆的邻近方法论

- 打开/关闭/保存 Document 的基础方法与事件（第1章）：本 skill 是其中"打开"在工作共享语境下的专项深化。

---

## 相关 skills

- revit-worksharing-api-decision-framework（composes-with）：打开之后的所有权操作（检出/放弃）见该 skill，与打开之时前后衔接。
- revit-transmissiondata-offline（contrasts-with）：两者都是"打开前预读文件信息"路径，但对象不同（工作集 vs 外部链接）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
