---
name: revit-synchronize-with-central
description: |
  编程执行"与中心同步"时调用：三个 API 分工——ReloadLatest（只拉取中心变更）/
  SynchronizeWithCentral（双向，无更改也执行"保存到中心"）/ HasAllChangesFromCentral（查询）。
  中心被锁定时默认等待重试，可经 SynchLockCallback 自定义（放弃等待立即返回）；
  SynchronizeWithCentralOptions 控制放弃内容与本地保存。Trigger："同步中心"、"中心模型锁定"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.13.6 与中心模型同步（约p406）
tags: [worksharing, synchronization, central-model, revit-api]
related_skills:
  - slug: revit-worksharing-api-decision-framework
    relation: composes-with
  - slug: revit-open-workshared-file
    relation: composes-with
  - slug: revit-serverpath-avoid
    relation: depends-on
---

# 与中心同步决策流程

## R — 原文 (Reading)

> "方法 Document.SynchronizeWithCentral() 重新载入来自中心模型的任何更改，以使当前会话是最新，然后当本地更改保存回中心。即使并未作更改，"保存到中心"也会执行。因同步需要临时锁定中心模型，若模型已被锁定则无法执行。"
>
> — 宦国胜，第5章 5.13.6 与中心模型同步 约p406

---

## I — 方法论骨架 (Interpretation)

中心-本地同步是一个"锁定-重试"模型下的三 API 决策。

- **三个 API 的分工**：
  - `ReloadLatest`：只读方向——把中心的最新变更拉到本地会话，不回写。
  - `SynchronizeWithCentral`：双向——先重载中心变更，再把本地变更保存回中心。注意语义：**即使本地无更改也会执行"保存到中心"**（刷新中心副本）。
  - `HasAllChangesFromCentral`：查询——本地是否已取全中心变更，用于"查询-再同步"的稳健模式（避免盲目同步）。
- **锁定行为**：同步需要临时锁定中心模型；中心已被他人锁定时，默认行为是**等待并多次尝试**。
- **自定义锁定策略**：构造 `TransactWithCentralOptions`，重写 `SynchLockCallback`——可在回调中选择放弃等待、立即返回，而不是无限等。
- **同步内容控制**：`SynchronizeWithCentralOptions.SetRelinquishOptions(...)` 决定同步后是否放弃已检出的图元/工作集；`SaveLocalAfterSync = false` 可跳过同步后的本地副本自动保存。
- 模式建议：先 HasAllChangesFromCentral 判断，需要时再 SynchronizeWithCentral，并显式配置锁定回调与放弃选项。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 中心文件被锁定时的同步

- **问题**: 同步时另一用户正锁定中心模型，插件卡在等待。
- **方法论的使用**: 按 5.13.6——默认等待多次重试；自定义则用 TransactWithCentralOptions 重写 SynchLockCallback，在回调中选择放弃等待立即返回。
- **结论**: 锁定行为是可编程控制的，不必接受默认无限等待。
- **结果**: 插件在锁定场景下快速返回并可稍后重试，不再挂起。

### 案例 2: 同步后不想自动保存本地副本

- **问题**: 默认同步后保存本地副本，批处理场景想省掉这一步。
- **方法论的使用**: SynchronizeWithCentralOptions 设 SaveLocalAfterSync=false，同时用 SetRelinquishOptions 控制放弃范围。
- **结论**: 同步的副作用（放弃所有权、保存本地）都是显式选项。
- **结果**: 批处理流程时间可控，所有权按配置精确放弃。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要在流程中自动执行"同步到中心"（批处理、定时任务）。
2. 同步时中心被锁，程序长时间挂起。
3. 只想拉取中心更新而不回写（预览他人变更）。

### 语言信号 (用户的话里出现这些就应激活)

- "SynchronizeWithCentral / ReloadLatest / 与中心同步"（sync with central）
- "中心模型 锁定 等待"（central model locked / waiting）
- "HasAllChangesFromCentral / SynchLockCallback"

### 与相邻 skill 的区分

本 skill 与 `revit-worksharing-api-decision-framework` 区分：那个管单会话内的检出/放弃，本 skill 管会话与中心之间的同步（同步也会触发放弃）。与 `revit-open-workshared-file` 构成"打开在前、同步在后"的协作链路。同步所依赖的 ModelPath 安全构造见 `revit-serverpath-avoid`。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **选择方向与 API**
   - 只拉取 → ReloadLatest；双向同步 → SynchronizeWithCentral（记住"无更改也保存到中心"）；不确定 → 先 HasAllChangesFromCentral。
   - 完成标准: API 选择有明确理由并写入注释/说明。

2. **配置锁定策略与同步选项**
   - 构造 TransactWithCentralOptions（SynchLockCallback：等待/放弃）；SynchronizeWithCentralOptions（RelinquishOptions、SaveLocalAfterSync）。
   - 完成标准: 锁定等待上限与放弃范围都是显式配置，无默认静默行为。
   - 判停条件: 只做只读拉取（ReloadLatest）→ 无需锁定配置，跳过此步。

3. **执行并处理结果**
   - 完成标准: 同步完成或因锁定快速返回，两条路径都有后续动作（重试/提示用户），不出现无限挂起。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单机文档——无中心概念，Save/SaveAs 即可。
- 交互式使用场景——让用户走 Revit 自带同步对话框更合适（含完整的用户确认流程），程序化同步用于自动化。

### 作者在书中警告的失败模式

- 中心被锁定时默认无限等待 → 程序挂起（本单元本体）。
- 以为无更改就不会写中心 → SynchronizeWithCentral 始终执行"保存到中心"。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版同步选项有扩展（如注释/视图集同步控制、WorksetConfiguration 相关刷新），"锁定-重试 + 回调定制"模型延续。

### 容易混淆的邻近方法论

- Document.Save/SaveAs：本地保存，不涉及中心锁定与双向交换。
- RelinquishOwnership（独立调用）：同步内的放弃是它的封装场景之一，可单独使用。

---

## 相关 skills

- revit-worksharing-api-decision-framework（composes-with）：同步内部会触发放弃（RelinquishOptions），是其批量放弃原则的一个应用点。
- revit-open-workshared-file（composes-with）：打开控制加载范围，同步控制双向交换与锁定策略，构成"打开在前、同步在后"的链路。
- revit-serverpath-avoid（depends-on）：同步的输入 ModelPath 须安全构造，避免 ServerPath 硬编码失效。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
