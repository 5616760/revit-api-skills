---
name: revit-failure-definition-registration
description: |
  发布自定义故障给 Revit 时使用。流程：OnStartup 注册到 FailureDefinitionRegistry → 运行时 PostFailure 拿 key → 条件满足 UnpostFailure(key) 撤销。何时调用：业务校验要提示用户；长事务中故障要自动消失。何时不调用：内置 BuiltInFailures 够用。Trigger：'发布自定义警告/PostFailure/UnpostFailure/注册故障'（post failure, failure definition registry, unpost failure）。文件必须可修改才能发布。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.1（约p366-367）
tags: [failure-definition, post-failure, registry, custom-warning, lifecycle]
related_skills:
  - slug: revit-failure-severity
    relation: depends-on
  - slug: revit-failures-processing-steps
    relation: composes-with
---

# 故障定义与注册流程（FailureDefinition）

## R — 原文 (Reading)

> 要使用故障发布机制来报告问题，请遵循以下步骤：(1)在外部应用程序的 OnStartup() 调用期间，Revit 尚未定义的新故障必须在 FailureDefinitionRegistry 中定义和注册。……
>
> — 宦国胜, 第5章 5.8.2 前置（约p366-367）

---

## I — 方法论骨架 (Interpretation)

Revit 的故障系统与常规 try-catch 完全不同：**故障要先"注册"，后"发布"**。发布故障等于把一条警告/错误放进事务的故障队列，由事务结束时的故障流水线决定怎么处理（弹窗、交给预处理器、自动解决……）。

完整生命周期分三步：

1. **注册（OnStartup，一次）**：自定义故障在 `OnStartup()` 里用 `FailureDefinitionRegistry` 定义并注册。需要指定 `FailureDefinitionId`（建议 GUID 保证唯一）、`FailureSeverity`（Warning/Error/DocumentCorruption，见 revit-failure-severity）、描述文本。
2. **发布（运行时）**：用 `FailureMessage` 相关类设置详细信息（元素引用、自定义文本），然后 `Document.PostFailure(failureMessage)` 把它投进当前事务。发布返回 `FailureMessageKey`。
3. **撤销（运行时，可选）**：在事务生存期内用 `Document.UnpostFailure(key)` 把已发布的故障撤掉——适合"故障随状态好转自动消失"的场景（如墙连接好了，警告就没了）。

两个硬约束：**文件必须可修改（有可写文档）才能发布故障**；故障在事务结束时由故障处理流水线（revit-failures-processing-steps）统一处理。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 重复出现的故障只提示一次、可自动撤销
- **问题**: V2 预测场景——一个故障可能在长事务中反复出现，想只提示一次、且能随状态好转自动撤销。
- **方法论的使用**: PostFailure 拿 key → 条件满足时 UnpostFailure(key) 消除已发布故障。
- **结论**: 注册（OnStartup 一次性）→ 运行时 PostFailure → 条件满足时 Unpost。
- **结果**: 用户只看到一次提示，状态好转后警告自动消失。

### 案例 2: 自定义故障 ID 的唯一性
- **问题**: 插件要定义自己的业务故障（如"房间未闭合"），不能用内置的 BuiltInFailures。
- **方法论的使用**: 用 FailureDefinition 类注册自定义故障，ID 用 GUID 保证唯一。
- **结论**: 自定义故障必须走注册流程，ID 全局唯一。
- **结果**: 故障可被稳定引用、撤销，不与内置故障冲突。

### 案例 3: 发布前确认文档可修改
- **问题**: 想发布故障但文档不可写。
- **方法论的使用**: 检查约束——文件必须可修改才能发布故障。
- **结论**: 发布前先确认文档可修改（如非只读、非链接文档）。
- **结果**: 避免在只读上下文里发布失败。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件校验出业务问题，要像 Revit 原生一样在模型里提示用户（而不是自己弹窗）。
2. 故障需要"状态好转自动消失"——按 key 撤销。
3. 想用 BuiltInFailures 之外的自定义故障。
4. 批处理里要控制"同一问题只提示一次"。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么给 Revit 发自定义警告？" / "post a custom warning in Revit"
- "PostFailure / UnpostFailure 怎么用？" / "how to use PostFailure and UnpostFailure"
- "自定义故障要注册吗？" / "do I need to register a custom failure"
- "FailureDefinitionId / FailureDefinitionRegistry" / "failure definition registration"
- "警告能自动消失吗？" / "can a warning disappear automatically"

### 与相邻 skill 的区分

注册自定义故障时，severity 等级选择由 `revit-failure-severity` 定义（本 skill 只做字段填写）；发布出去的故障在事务结束时由 `revit-failures-processing-steps` 三步流水线消费——本 skill 是"生产故障"，那个是"消费故障"。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **注册故障定义**：在 `IExternalApplication.OnStartup()` 里用 FailureDefinitionRegistry 注册自定义故障（FailureDefinitionId + FailureSeverity + 描述文本）。
   - 完成标准: 注册成功；ID 唯一（GUID）；severity 明确。

2. **运行时发布**：
   - 确认文档可修改。
   - 构造 FailureMessage（设置元素引用、自定义文本等）。
   - `FailureMessageKey key = doc.PostFailure(msg);` 保存 key。
   - 完成标准: PostFailure 成功返回 key。

3. **按需撤销并收尾**：
   - 条件满足时 `doc.UnpostFailure(key);` 让故障消失。
   - 在事务结束前完成发布/撤销，让流水线正确处理。
   - 完成标准: 故障在正确时机出现/消失；未出现"发布后无法撤销"的悬挂状态。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 内置 BuiltInFailures 已有对应故障——直接用，不必自定义。
- 不需要撤销、一次性提示——可以直接发布，不用管 key 生命周期。
- 文档不可修改（只读/链接）——不能发布，改用别的方式提示。

### 作者在书中警告的失败模式

- **新故障必须在 OnStartup 注册**——运行时才注册会失败；漏注册 = 发布失败。
- **文件必须可修改才能发布**——只读上下文发布失败。
- **DocumentCorruption 级故障发布时不能从文件读信息**（revit-failure-severity）——发布该级要非常小心。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 FailureDefinition 的 API 形态（Builder 模式 vs 传统构造）有变化，应按当前 SDK 写法实现。
- 未讨论故障消息的本地化、多语言展示细节。

### 容易混淆的邻近方法论

- "注册"（OnStartup 一次） vs "发布"（运行时多次）：注册是定义，发布是使用——别在运行时反复注册。
- `FailureMessageKey` vs `FailureDefinitionId`：前者是本次发布实例的钥匙（用于 Unpost），后者是故障类型的全局身份（用于识别）——两个不同维度的 ID。

---

## 相关 skills

- revit-failure-severity（depends-on）：注册时要选严重程度等级，severity 语义由该 skill 定义。
- revit-failures-processing-steps（composes-with）：发布出去的故障由三步流水线消费，本 skill 是故障生产端。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
