---
name: revit-iupdater-execute-transaction-rules
description: |
  编写 IUpdater.Execute()（DMU）内部逻辑时调用。规则：Execute 在事务结束、DocumentChanged 之前
  调用；禁止开新 Transaction（抛异常），可用 SubTransaction；修改自动并入原始事务（随其回滚）；
  外部数据更新不随原事务回滚，需自行补偿。GetAdded/Deleted/ModifiedElementIds 是识别触发图元的
  入口。不适用于事后通知（用 DocumentChanged）。Trigger："IUpdater/DMU"、"Execute 里开事务报错"、
  "自动联动修改"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.6.2（约p359-360）
tags: [dmu, iupdater, transactions, revit-api]
related_skills:
  - slug: revit-iupdater-forbidden-api-list
    relation: composes-with
  - slug: revit-transaction-hierarchy
    relation: depends-on
---

# IUpdater Execute 方法约束与事务规则

## R — 原文 (Reading)

> "Revit 在文件事务结束时调用该方法……更新器是在 DocumentChanged 事件之前被调用的，因此该事件将包含所有更新器所作出的更改。在实现此方法时，将无法打开任何新事务（会引发异常），但如果需要的话，可以使用子事务。虽然也可以用它来更新文件的外部数据，但这种更改不会成为原始事务的一部分……"
>
> — 宦国胜，第5章 5.6.2（约p359-360）

---

## I — 方法论骨架 (Interpretation)

IUpdater.Execute 是一个"嵌在别人事务里"的特殊执行环境，理解这一点是写对更新器的前提。

- 执行时机：文件事务结束时被调用，且**早于 DocumentChanged**——所以 DocumentChanged 能看到更新器造成的所有修改。
- 事务禁令：Execute 内不能开新 Transaction（抛异常）；需要中途提交/部分可见时只能用 SubTransaction。
- 并入语义：Execute 中的模型修改自动并入原始事务——用户撤销时会连同更新器的修改一起回滚，这是"联动一致性"的来源。
- 外部数据边界：Execute 里写文件/数据库的修改**不会**随原始事务回滚。若要求一致性，必须在 Execute 内自己写补偿逻辑（检测原事务失败并回滚外部写入）。
- 触发图元识别：`UpdaterData.GetAddedElementIds()` / `GetDeletedElementIds()` / `GetModifiedElementIds()` 三兄弟是标准入口。
- 选型判断：分析结果自动刷新等"模型变更即重算"的需求，书里明确推荐 DMU（见 5.11.2）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 更新器内的修改需要事务保护

- **问题**: 更新器逻辑需要分阶段提交，但开新 Transaction 抛异常。
- **方法论的使用**: 用 SubTransaction 做细粒度控制；Execute 内的修改自动并入原始事务，天然受撤销/重做保护。
- **结论**: 更新器永远工作在"借用外层事务"的模式下。
- **结果**: 修改与用户操作在同一个撤销单元里，Ctrl+Z 一起回滚。

### 案例 2: 更新器同时写外部数据库

- **问题**: 更新器既要改模型又要同步外部数据，用户撤销时模型回滚了但数据库没回滚。
- **方法论的使用**: 书中指出外部数据更改不会成为原始事务的一部分；需要一致性时在 Execute 内实现补偿逻辑。
- **结论**: "模型并入、外部自管"是硬边界，一致性成本由开发者承担。
- **结果**: 在补偿逻辑覆盖后，撤销操作后模型与外部数据重新一致。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 实现自动联动规则：改墙厚自动改类型、删梁自动删关联标注等。
2. 在 Execute 里开 Transaction 报错，或撤销后行为不一致。
3. 更新器里要同时更新外部系统（数据库/文件）。

### 语言信号 (用户的话里出现这些就应激活)

- "IUpdater / DMU / dynamic model update"
- "Execute 里开事务 异常"（cannot open transaction in updater）
- "模型变了自动修改其他图元"（auto-update linked elements）
- "GetModifiedElementIds / UpdaterData"

### 与相邻 skill 的区分

- 与 `revit-iupdater-forbidden-api-list` 的关系：本 skill 讲 Execute 的事务规则（禁新事务/可用子事务）；该 skill 讲具体禁入 API 清单，两者同属 Execute 安全边界、维度互补。
- 与 `revit-transaction-hierarchy` 的关系：更新器 Execute 内的事务约束建立在对通用事务层级（Transaction/TransactionGroup）的理解之上。
- 与 `revit-documentclosing-no-model-edit` 的区别：事件回调里改模型非法；更新器 Execute 里改模型合法且推荐，关键是载体选择。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **从 UpdaterData 提取触发图元**
   - 用 GetAdded/Deleted/ModifiedElementIds 拿到变更集合，先判断非空再处理。
   - 完成标准: 不假设"只有一个变更"，循环处理集合。

2. **在 Execute 内写修改，遵守事务规则**
   - 直接修改（自动并入原事务）；需要分段用 SubTransaction；全程无 `new Transaction(...)`。
   - 完成标准: 代码中搜索不到 Execute 路径上的 Transaction.Start；有外部写入时附补偿逻辑设计。
   - 判停条件: 若发现需求本质是"事后通知/审计"而非"联动修改"，停止用更新器，改用 DocumentChanged 订阅。

3. **验证撤销一致性**
   - 完成标准: 用户撤销触发操作后，更新器修改一并回滚；外部数据有对应补偿。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 外部数据同步为主、不改模型 → DocumentChanged（只读）更简单、更安全。
- 需要用户交互确认的修改 → 更新器无 UI，改用 ExternalEvent 或命令。

### 作者在书中警告的失败模式

- Execute 内开新 Transaction → 异常（本单元本体）。
- 依赖外部数据随原事务回滚 → 不会发生，产生模型/数据不一致。
- 调用禁入 API（Save、LoadFamily、ViewSheet.AddView 等）→ 见 `revit-iupdater-forbidden-api-list`。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版新增了 ChangePriority 等调度细节与部分放宽，但"禁新事务、并入原事务"的骨架未变。

### 容易混淆的邻近方法论

- SubTransaction/TransactionGroup 层级语义（第5章 5.9）：更新器内只能用最内层那级。
- DocumentChanged 的"事后只读" vs 更新器的"事中可写"：一字之差，职责完全不同。

---

## 相关 skills

- **revit-iupdater-forbidden-api-list**（IUpdater Execute 中禁止调用的 API 清单 · composes-with）— 该 skill 提供 Execute 内禁止调用的 API 清单，与事务规则同属 Execute 安全边界。
- **revit-transaction-hierarchy**（事务三件套层级与组合 · depends-on）— 更新器事务规则依赖通用事务三件套（Transaction/TransactionGroup）的层级知识。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
