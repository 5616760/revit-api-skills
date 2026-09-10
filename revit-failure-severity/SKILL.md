---
name: revit-failure-severity
description: |
  为自定义故障选严重程度时使用。三级：Warning 用户可忽略；Error 不可忽略，强烈建议至少一个解决方案；DocumentCorruption 文件损坏专用、强制事务尽快回滚。何时调用：注册故障定义、设计 Resolution。何时不调用：用内置 BuiltInFailures。Trigger：'故障严重程度选哪个/Warning 还是 Error/DocumentCorruption'（failure severity, forced rollback）。DocumentCorruption 不能读文件信息、须先回滚、不加 Resolution。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.1（约p367-368）
tags: [failure-severity, warning, error, document-corruption, failure-resolution]
related_skills:
  - slug: revit-failures-accessor
    relation: composes-with
---

# 故障严重程度分级原则（Warning / Error / DocumentCorruption）

## R — 原文 (Reading)

> Warning：最终用户可以忽略的故障……Error：不可忽略的故障……强烈建议，每个这种严重程度的故障至少有一个解决方案。DocumentCorruption：由已知的文件损坏所造成的故障。此故障会强制 Transaction 尽快回滚。
>
> — 宦国胜, 第5章 5.8.1 严重程度（约p367-368）

---

## I — 方法论骨架 (Interpretation)

Revit 的故障不是简单的"错误弹窗"，而是一种**事务控制信号**——严重程度直接决定事务的命运：

- **Warning（警告）**：用户可忽略的故障。事务可以正常提交，警告只作为提示存在。
- **Error（错误）**：不可忽略的故障。事务不能假装没事——强烈建议每个 Error 至少配一个**解决方案（FailureResolution）**，让用户/处理器能选"怎么解决它"（如"删除图元"、"忽略"）。没有解决方案的 Error，处理体验会很差。
- **DocumentCorruption（文件损坏）**：由**已知的文件损坏**造成的故障。它是**强制信号**——事务必须**尽快回滚**。规则：
  - 发布该级故障时**不能从文件读任何信息**（文件已损坏，读也是坏的）；
  - **须先回滚当前事务**再发布；
  - 该级**不能添加 FailureResolution**（事务都回滚了，谈不上"解决"）。
  - 平时应避免使用——能用 Error + Resolution 解决的就别用 DocumentCorruption。

附加机制：故障可以有多个解决方案时，`SetDefaultResolutionType()` 可指定默认方案；否则**第一个添加的方案自动成为默认**。

方法论本质：**选 severity 就是选"这个故障允许事务怎么走"**——Warning 放行、Error 裁决、DocumentCorruption 强制回滚。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 数据不一致、必须中断当前事务
- **问题**: V2 预测场景——想发布一个"数据不一致、必须中断当前事务"的故障，且不能从文件读任何信息，选哪级？
- **方法论的使用**: 匹配约束——不能读文件信息 + 强制回滚 = DocumentCorruption。
- **结论**: DocumentCorruption 是"文件损坏"专用级，强制 Transaction 尽快回滚。
- **结果**: 发布时不能从文件读信息、须先回滚当前事务；该级不能添加 FailureResolution。

### 案例 2: Error 必须配解决方案
- **问题**: 自定义故障是不可忽略的 Error，用户/处理器怎么处理它？
- **方法论的使用**: 强烈建议每个 Error 至少有一个 FailureResolution。
- **结论**: Error 不能"干抛"——要有可选的解决动作。
- **结果**: 故障处理流水线（revit-failures-accessor）里 ResolveFailure 才有东西可解决。

### 案例 3: 默认解决方案选择
- **问题**: 一个故障有多个解决方案，默认用哪个？
- **方法论的使用**: SetDefaultResolutionType() 可改默认方案，否则第一个添加的方案自动成默认。
- **结论**: 显式指定默认方案，避免"第一个"隐含语义造成意外。
- **结果**: 用户看到的首选方案符合预期。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 注册自定义故障时不确定选 Warning / Error / DocumentCorruption。
2. 需要发布"必须中断事务"级别的故障。
3. 设计故障的解决方案（Resolution）列表。
4. 理解"为什么这个故障事务被强制回滚了"。

### 语言信号 (用户的话里出现这些就应激活)

- "故障严重程度怎么选？" / "which failure severity should I use"
- "Warning 和 Error 有什么区别？" / "difference between Warning and Error"
- "DocumentCorruption 什么时候用？" / "when to use DocumentCorruption"
- "这个故障必须回滚事务" / "force the transaction to roll back"
- "Error 需要解决方案吗？" / "does Error need a FailureResolution"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-accessor` 配合：severity 决定故障如何分类，accessor 决定已分类故障如何处置。与 `revit-failure-definition-registration` 的关系：注册流程里要填 severity 字段，本 skill 是选值之前的决策知识。异常（`revit-exception-types`）是代码层机制，与故障（事务层）两套体系，severity 只属于故障体系。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断故障性质**：问三个问题——用户能忽略吗？事务还能安全继续吗？根因是不是文件损坏？
   - 可忽略 → Warning。
   - 不可忽略但事务可继续/可解决 → Error。
   - 文件已损坏、必须中止 → DocumentCorruption。
   - 完成标准: severity 选定且有明确理由。

2. **按等级补齐要素**：
   - Warning：直接发布即可。
   - Error：**为它添加至少一个 FailureResolution**；多个方案时用 SetDefaultResolutionType 指定默认。
   - DocumentCorruption：确认当前事务可回滚；发布前/时**不从文件读信息**；**不添加 Resolution**。
   - 完成标准: 该等级要求的要素齐备（Error 有 Resolution，DocumentCorruption 无 Resolution 且回滚前置）。

3. **注册并验证**：
   - 在 OnStartup 注册故障定义（含 severity）。
   - 触发一次，观察事务行为符合等级语义（Warning 放行 / Error 裁决 / Corruption 回滚）。
   - 完成标准: 事务行为与所选等级一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 内置 BuiltInFailures 已有对应故障——severity 已内置，直接用。
- 不需要用户决策的提示——考虑用 Warning 或日志，别滥用 Error 打断。

### 作者在书中警告的失败模式

- **DocumentCorruption 是"文件损坏"专用**——不是普通严重错误的升级版，滥用会导致文件被强制回滚。
- **DocumentCorruption 不能读文件信息、须先回滚、不能加 Resolution**——三条硬约束都违反就是踩雷。
- **Error 没有解决方案**——用户/处理器无路可走，体验与收敛性都受损。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 severity 枚举与 DocumentCorruption 的触发行为细节有演进，应按当前 SDK 核对。
- 未讨论"DocumentCorruption 在实际文件中如何触发/恢复"的工程细节——此类故障应配合日志与文件修复流程。

### 容易混淆的邻近方法论

- Warning vs Error：区分标准是"能不能忽略"——不是"重不重要"。
- Error + Resolution vs DocumentCorruption：前者"解决后事务可继续"，后者"事务必死"——选择前先问"事务还能救吗"。

---

## 相关 skills

- revit-failures-accessor（composes-with）：severity 决定故障分类，accessor 决定处置手段，二者配合处理已分级故障。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
