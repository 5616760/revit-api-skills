---
name: revit-readonly-event-checks
description: |
  在任何 Revit 事件回调里要修改模型之前调用：先检查 Document.IsModifiable 与
  Document.IsReadOnly，确认文档当前可编辑再动手；API 文档标注的只读事件是一层约束，但常规
  事件里文档也可能不可编辑（工作共享只读检出、被其他命令占用等）。"事件 ≠ 可写"是统一认知。
  不满足时放弃修改或改用 IUpdater/ExternalEvent 延后。Trigger："事件里改图元"、"IsModifiable"、
  "IsReadOnly"、"文档不可编辑异常"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.3（约p349-350）
tags: [events, readonly, defensive-checks, revit-api]
related_skills: []
---

# 只读事件与模型修改检查原则

## R — 原文 (Reading)

> "某些事件被视为只读事件，这意味着在执行期间不得修改模型。程序员应检查 Document.IsModifiable 和 Document.IsReadOnly 属性，以确定该模型是否可被修改。"
>
> — 宦国胜，第5章 5.3（约p349-350）

---

## I — 方法论骨架 (Interpretation)

这个原则把"能不能改模型"从"查文档背清单"变成"运行时检查"，是事件回调的安全底线。

- 两层约束模型：
  1. **静态层**：API 文档标注的只读事件（回调期间禁止修改）——这是事件类型的固有属性。
  2. **动态层**：即使是常规事件，文档当时也可能不可编辑——工作共享下图元被别人检出、文档被其他操作占用等。
- 检查方法：动手前读 `Document.IsModifiable`（当前是否处于可修改状态）与 `Document.IsReadOnly`（文档是否只读）。
- 决策分支：
  - 两项都通过 → 可以修改（在允许写的事件里）。
  - 任一不通过 → 放弃本次修改，或改用 IUpdater（事务中联动）/ ExternalEvent（延后到空闲周期）。
- 统一口诀："事件 ≠ 可写"。文档没说只读不等于当下能改；文档说只读则一定不能改。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: DocumentSaving 里改图元

- **问题**: DocumentSaving 看起来不是只读事件，回调里直接改图元。
- **方法论的使用**: 按两层约束模型检查——事件本身可能允许写，但保存过程中文档状态未必可编辑；先查 IsModifiable/IsReadOnly。
- **结论**: "看起来可写"不构成动手依据，运行时状态才是。
- **结果**: 加检查后，不可写分支被安全跳过，不再抛异常。

### 案例 2: 工作共享只读检出下的回调

- **问题**: 常规事件回调中，目标图元被其他用户检出，修改失败。
- **方法论的使用**: IsReadOnly/IsModifiable 检查发现不可编辑，转用 ExternalEvent 延后或提示用户先检出。
- **结论**: 检查原则把"随机崩溃"变成"可预期分支"。
- **结果**: 工作共享环境下插件行为稳定。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在任意事件回调中写修改代码，做防御性设计。
2. 插件在多用户工作共享项目中偶发"文档只读/不可编辑"异常。
3. 事件回调偶发失败但本地单机正常，怀疑环境差异。

### 语言信号 (用户的话里出现这些就应激活)

- "事件处理程序里修改模型"（modify model in event handler）
- "IsModifiable / IsReadOnly 检查"
- "工作共享 只读 检出 异常"（workshared checked-out readonly exception）

### 与相邻 skill 的区分

- 与 `revit-documentclosing-no-model-edit` 的区别：该 skill 是具体反例（只读事件必炸）；本 skill 是通用原则——任何事件回调动手前都做 IsModifiable/IsReadOnly 运行时检查，覆盖动态不可编辑状态。
- 与 `revit-iupdater-execute-transaction-rules` 的区别：更新器 Execute 里不需要也不允许自己开事务；本原则针对的是事件回调场景。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **静态判断：查事件只读性**
   - 查 API 文档确认目标事件是否只读事件。
   - 完成标准: 记录"只读/常规"结论。
   - 判停条件: 只读事件 → 直接进入第 3 步选替代载体，跳过修改尝试。

2. **动态检查：IsModifiable + IsReadOnly**
   - 在修改代码前 `if (!doc.IsModifiable || doc.IsReadOnly) { ... }`。
   - 完成标准: 所有修改路径都有该守卫，不满足时走降级分支。

3. **选替代载体**
   - 联动修改 → IUpdater；延后执行 → ExternalEvent；提示用户 → UI 反馈。
   - 完成标准: 不可写场景有明确出口，无"硬闯"路径。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 外部命令 Execute 内——命令本身提供可写上下文（仍建议检查 IsReadOnly，但 IsModifiable 恒真）。
- 只读操作（查询、统计）——不受此原则约束。

### 作者在书中警告的失败模式

- 只信 API 文档的只读清单，忽略运行时状态 → 常规事件里偶发崩溃。
- 检查失败后强行修改 → 异常中断整个回调。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版对部分事件开放了受限写能力，但"先检查再动手"的原则在所有版本都成立。

### 容易混淆的邻近方法论

- IUpdater / ExternalEvent：检查失败后的两个标准替代载体，注意三者职责分工。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
