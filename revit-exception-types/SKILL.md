---
name: revit-exception-types
description: |
  Revit 专有异常时使用。所有 API 异常继承 ApplicationException；专有异常 AutoJoinFailedException/RegenerationFailedException/ModificationOutsideTransactionException 直接编码 Revit 约束，InternalException 是非预期故障路径。何时调用：Execute 入口 try-catch 兜底、判断根因。Trigger：'专有异常/ModificationOutsideTransactionException'（Revit specific exception）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.3.7（约p044）
tags: [exception, error-handling, try-catch, api-errors, transaction-mode]
related_skills:
  - slug: revit-failures-processing-steps
    relation: contrasts-with
  - slug: revit-failure-severity
    relation: contrasts-with
  - slug: revit-regenerate-failure-rollback
    relation: composes-with
---

# Revit 异常体系与专有异常类型

## R — 原文 (Reading)

> 然而，尚有一些 Revit 独有的异常子类：AutoJoinFailedException、RegenerationFailedException、ModificationOutsideTransactionException。此外还有一个称作 InternalException 的特殊异常类型。
>
> — 宦国胜, 第1章 1.3.7（约p044）

---

## I — 方法论骨架 (Interpretation)

Revit 的异常体系分两层，理解后就能快速读懂"这个异常在告诉我什么"：

1. **通用层**：所有 API 异常都继承自 `Autodesk.Revit.Exceptions.ApplicationException`，它又继承 .NET 的 `Exception`。所以 catch `Exception ex` 永远能兜底。
2. **专有层**：Revit 定义了一批"名字即约束"的异常子类，每个都对应一条 Revit 特有的规则——
   - `ModificationOutsideTransactionException`：你在事务外改模型了（最常见的新手错误）。
   - `RegenerationFailedException`：重生成失败，需要回滚（见 revit-regenerate-failure-rollback）。
   - `AutoJoinFailedException`：自动连接失败。
   - `InternalException`：非预期的内部故障路径——不该走到这里的内部错误。

方法论要点：**用异常类型做第一层诊断**。看到 `ModificationOutsideTransactionException`，先查"是不是没包事务/事务模式是不是 ReadOnly"；看到 `RegenerationFailedException`，先查"Regenerate 后有没有处理回滚"。同时按第1章 1.2.2 的做法，在外部命令 `Execute` 入口用 try-catch 兜底，避免让异常直接冒泡到 Revit 用户面前。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: ReadOnly 事务模式下修改参数
- **问题**: V2 预测场景——在 ReadOnly 事务模式下尝试修改图元参数，会抛什么异常？
- **方法论的使用**: ReadOnly 模式不创建事务，而所有模型修改必须在事务中——违反约束必抛专有异常。
- **结论**: 抛 ModificationOutsideTransactionException。
- **结果**: 这是最常见的新手错误之一，异常名直接告诉你"你不在事务里"。

### 案例 2: Execute 入口的兜底捕获
- **问题**: 第1章 1.2.2 外部命令里 try-catch 的应用。
- **方法论的使用**: 在 Execute 入口用 try-catch(Exception ex) 兜底捕获，把异常转成对用户友好的错误消息。
- **结论**: 通用异常 + 专有异常都逃不出 ApplicationException 这一脉，catch 基类即可。
- **结果**: 插件在异常时不崩溃、不留半完成状态，用户得到明确提示。

### 案例 3: 事务模式选择与异常的联动
- **问题**: 第1章 1.3.6 事务模式（revit-transaction-mode-selection）与异常的关系。
- **方法论的使用**: TransactionMode 决定框架是否自动开事务，直接决定你会不会踩 ModificationOutsideTransactionException。
- **结论**: Automatic 模式框架管事务，ReadOnly 模式不建事务、改模型必抛异常。
- **结果**: 选错模式 = 批量异常，靠异常类型能快速定位根因。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件运行抛了看不懂的 Revit 专有异常，需要判断含义与修复方向。
2. 写外部命令时不确定要不要 try-catch、catch 什么。
3. 新手报"ModificationOutsideTransactionException"——很可能是没包事务或事务模式选错。
4. 需要在错误消息里区分"代码 bug"与"API 约束违规"。

### 语言信号 (用户的话里出现这些就应激活)

- "ModificationOutsideTransactionException 是啥？" / "ModificationOutsideTransactionException thrown"
- "RegenerationFailedException 怎么处理？" / "handle RegenerationFailedException"
- "AutoJoinFailedException 什么时候抛？" / "when is AutoJoinFailedException thrown"
- "Revit 有哪些专有异常？" / "Revit specific exception types"
- "Execute 里要不要 try-catch？" / "should I try-catch in Execute"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-processing-steps`、`revit-failure-severity` 区分：异常（.NET 代码层抛出机制）与故障（Revit 事务收尾裁决机制）是两套体系，severity 分级也属于故障系统，本 skill 只讲异常谱系。与 `revit-regenerate-failure-rollback` 配合：RegenerationFailedException 的恢复契约由它专管，本 skill 负责定位异常含义与修复方向。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **识别异常类型**：查看抛出的异常完整类型名（不要只看 ex.Message）。
   - 完成标准: 确认异常是否继承 Autodesk.Revit.Exceptions.ApplicationException，以及具体子类。

2. **按类型定向修复**：
   - `ModificationOutsideTransactionException` → 查是否漏包 Transaction 或 TransactionMode 为 ReadOnly。
   - `RegenerationFailedException` → 查 Regenerate 后是否有回滚（见 revit-regenerate-failure-rollback）。
   - `AutoJoinFailedException` → 查自动连接相关操作/元素。
   - `InternalException` → 视为 Revit 内部 bug 或意外路径，记录现场并最小化复现。
   - 其他通用异常 → 按 .NET 常规处理。
   - 完成标准: 每个异常都归到明确根因并给出修复动作。

3. **在 Execute 入口加兜底**：用 try-catch(Exception ex) 包住命令主逻辑，异常转为 Result.Failed + 用户可读消息。
   - 完成标准: 命令入口不再有未捕获异常冒泡；错误消息能指导修复。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 异常已在事务/故障处理层被妥善处理——不需要重复兜底。
- 用户只是想看错误文案、不涉及类型判断。

### 作者在书中警告的失败模式

- **所有 API 方法都抛 ApplicationException 子类**——别只 catch 某个精确类型而漏掉其他同族异常；兜底要 catch 基类。
- **InternalException 表示非预期故障路径**——别当成普通异常忽略，它通常意味着 Revit 内部状态异常。
- **ModificationOutsideTransactionException 是新手高频异常**——在事务模式/事务包裹上出问题几乎必抛它。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：后续版本新增/调整了若干异常子类（如族加载、协同相关的异常），应以当前 SDK 文档为准。
- 未讨论 .NET 的 `AggregateException`、`TaskCanceledException` 等异步相关异常在 Revit 插件里的传播——现代插件 async 化后这是新坑。

### 容易混淆的邻近方法论

- 异常（Exception，代码抛出机制） vs 故障（Failure，事务提交时的裁决机制）：两者都叫"错误处理"，但处理入口完全不同——异常用 try-catch，故障用 Preprocessor/Processor 流水线。
- `ModificationOutsideTransactionException` vs `RegenerationFailedException`：前者是"不该写却写了"，后者是"该重生成却失败了"——一个是上下文违规，一个是过程失败。

---

## 相关 skills

- revit-failures-processing-steps（contrasts-with）：异常是代码层抛出机制，故障是事务收尾裁决机制，两套错误处理体系勿混用。
- revit-failure-severity（contrasts-with）：故障严重程度分级属于故障系统，本 skill 讲的是异常谱系。
- revit-regenerate-failure-rollback（composes-with）：RegenerationFailedException 的恢复契约由它专管，本 skill 是其诊断入口。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
