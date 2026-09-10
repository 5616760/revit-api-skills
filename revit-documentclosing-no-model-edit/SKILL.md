---
name: revit-documentclosing-no-model-edit
description: |
  在 Revit 事件回调（尤其 DocumentClosing、DocumentChanged 等只读事件）中尝试修改模型、遇到
  "事件里能改模型吗"问题时调用。结论：只读事件中修改必抛异常，联动修改须迁到 IUpdater，外部
  同步只能只读。不适用于普通命令 Execute 内的模型修改（合法事务上下文）。Trigger：事件处理程序
  报异常、"关闭文档前自动保存/同步"（before close auto-save / sync in event）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.2.1（约p329-330）
tags: [events, readonly-events, counter-example, revit-api]
related_skills:
  - slug: revit-iupdater-execute-transaction-rules
    relation: contrasts-with
  - slug: revit-readonly-event-checks
    relation: composes-with
  - slug: revit-external-events-nonmodal-dialog
    relation: composes-with
---

# 反例：DocumentClosing 事件处理程序中尝试修改文件

## R — 原文 (Reading)

> "要注意的是，在某些事件（如 DocumentClosing 事件）过程中，活动文件是不允许修改的。在这样的事件处理过程中，如果事件处理程序试图修改，则将引发异常。"
>
> — 宦国胜，第5章 5.2.2.1（约p329-330）

---

## I — 方法论骨架 (Interpretation)

这是一个"在错误的地方做正确的事"的反例。方法论核心是：**Revit 的事件回调不等于可写上下文**。

- Revit 把事件分为两类：只读事件（回调期间禁止修改模型）和常规事件（可能允许，但不保证）。
- DocumentClosing 属于只读事件：文档正在关闭，模型已进入不可写状态，此时开事务或改图元会直接抛异常。
- 常见的错误动机是"趁关闭前做点收尾修改"（自动保存、数据回写、清理图元）——这条路在事件回调里走不通。
- 正确的分流是：
  1. 要"模型一变就联动修改其他图元"→ 用 IUpdater（动态模型更新），它在 DocumentChanged 之前执行，修改自动并入原事务。
  2. 要"只同步数据到外部"（数据库、文件）→ 在 DocumentChanged 里做，但严格只读。
  3. 要"交互式修改"→ ExternalEvent / 外部命令。
- 判断口诀：先问"这是只读事件吗"，再问"我的操作是读还是写"，两个都不是只读才能动手。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 关闭文档前自动把图元数据同步到外部数据库

- **问题**: 开发者想在 DocumentClosing 回调中开新事务，把模型数据写回外部数据库并顺手修正图元。
- **方法论的使用**: 书中指出 DocumentClosing 是只读事件，回调中修改活动文件会引发异常；数据同步类需求应放到只读语义下执行，模型修改类需求必须换载体。
- **结论**: 事件回调里只能做只读操作；"改模型"必须迁到 IUpdater，"同步外部数据"可在事件里只读完成。
- **结果**: 遵循该职责划分的代码不会在关闭流程中抛异常；违规代码在触发事件时直接崩溃。

### 案例 2: 在 DocumentChanged 里改模型

- **问题**: 想实现"模型一变就自动修改其他图元"，直觉做法是在 DocumentChanged 回调里开事务改模型。
- **方法论的使用**: 书中说明 DocumentChanged 是事后只读通知，回调内开新事务违反只读语义；正确载体是 IUpdater（先于 DocumentChanged 执行，修改并入原事务）。
- **结论**: "通知类"事件只用于观察，"联动修改"交给更新器。
- **结果**: 用 IUpdater 的实现可在撤销/重做中保持一致；在事件里改模型的实现进入异常路径。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在 Revit 插件事件处理程序中调用修改 API 时抛出异常，搜到"只读事件"相关报错。
2. 想实现"关闭/保存文档前自动做某事"（自动保存、数据回写、日志、清理）。
3. 想实现"模型一变化就自动更新其他图元"的联动逻辑。

### 语言信号 (用户的话里出现这些就应激活)

- "DocumentClosing 事件里怎么修改模型 / 为什么抛异常"
- "关闭文档前自动保存 / 同步数据库"（auto-save before close / sync in event handler）
- "DocumentChanged 里能改图元吗"（modify model in DocumentChanged）
- "事件处理程序 事务 异常"（event handler transaction exception）

### 与相邻 skill 的区分

- 与 `revit-readonly-event-checks` 的关系：本 skill 是具体反例——DocumentClosing/DocumentChanged 等只读事件中修改必然失败；该 skill 是“回调动手前做 IsModifiable/IsReadOnly 检查”的通用原则。
- 与 `revit-iupdater-execute-transaction-rules` 的区别：该 skill 是“联动修改的正确载体”（更新器 Execute）的正面框架；本 skill 是“错误载体”（只读事件）的反面教材，两者正反配对。
- 与 `revit-external-events-nonmodal-dialog` 的关系：需要事件驱动的模型修改时，应改投 ExternalEvent 框架。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认事件类型**
   - 查该事件是否属于 API 文档标注的只读事件（DocumentClosing、DocumentChanged 等）。
   - 完成标准: 明确写出"该事件只读/常规"的结论及依据。

2. **把需求按"读/写"分流**
   - 写模型 → IUpdater（注册触发器，Execute 内做修改）；只读同步 → 留在事件回调；交互触发 → ExternalEvent。
   - 完成标准: 每个原始需求都映射到一个合法载体，事件回调内不再有事务/修改调用。
   - 判停条件: 若需求本质上必须在关闭瞬间改模型，则告知用户此路不通，改走"关闭前提示用户手动执行命令"或文档打开时的对账逻辑。

3. **验证回调内代码全部只读**
   - 完成标准: 回调内无 Transaction/SubTransaction、无图元修改 API；外部数据写入不依赖模型可写状态。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 外部命令 Execute 里的正常模型修改——那是合法事务上下文，与事件无关。
- 非模态对话框触发修改——那是 ExternalEvent 场景，不是本反例。

### 作者在书中警告的失败模式

- 只读事件处理程序中试图修改活动文件 → 引发异常（本单元本体）。
- DocumentChanged 内开新事务改模型 → 违反只读语义，进入"事件中事务"复杂路径。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 API；后续版本对事件可写性有局部调整（如部分版本允许特定事件内开事务但仍不推荐），以当前版 API 文档为准。

### 容易混淆的邻近方法论

- IExternalEventHandler / ExternalEvent：异步投递修改的框架，不是事件回调内直接改。
- DocumentChanged 只读通知 vs IUpdater 联动修改：职责边界见 `revit-iupdater-execute-transaction-rules`。

---

## 相关 skills

- **revit-iupdater-execute-transaction-rules**（IUpdater Execute 方法约束与事务规则 · contrasts-with）— 事件回调里改模型非法，更新器 Execute 里改模型合法且推荐——二者构成正反配对。
- **revit-readonly-event-checks**（只读事件与模型修改检查原则 · composes-with）— 本 skill 是只读事件修改必败的具体反例，与该 skill 的通用检查原则互补。
- **revit-external-events-nonmodal-dialog**（External Events 框架实现非模态对话框 · composes-with）— 事件回调的模型修改诉求应转投该 skill 的 ExternalEvent 框架在空闲周期执行。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
