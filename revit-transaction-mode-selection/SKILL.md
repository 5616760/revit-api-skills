---
name: revit-transaction-mode-selection
description: |
  当给外部命令类加 [Transaction(TransactionMode.XXX)]、纠结 Automatic/Manual/ReadOnly 时调用。决策树：要写模型？否→ReadOnly；是→需
  精细事务控制？否→Automatic；是→Manual。Automatic 按返回值自动提交/回滚；Manual 自管事务；ReadOnly 禁止创建事务。不适用于：事务内部逻辑。trigger：Tr
  ansactionMode、事务模式、ReadOnly 报错、transaction mode。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p043
tags: [transaction, external-command, attribute, revit-api]
related_skills:
  - slug: revit-command-result-undo
    relation: composes-with
  - slug: revit-transaction-hierarchy
    relation: composes-with
---

# TransactionMode 选择决策框架

## R — 原文 (Reading)

> TransactionMode.Automatic：外部命令执行前，Revit 将在活动文件中创建一个事务，命令完成后事务将被提交或回滚（基于 ExternalCommand 回调函数的返回值）。该命令无法创建和启动自身事务，但可以创建子事务。
>
> — 宦国胜, 第1章 1.3.6（约 p043）

---

## I — 方法论骨架 (Interpretation)

Revit 要求你**在类上声明**事务模式，而不是在代码里临时决定。这个声明通过 `[Transaction(TransactionMode.XXX)]` 属性完成，三种模式把"谁来管理事务、允许做什么"划分成三个档位：

- **Automatic（自动档）**：Revit 在执行命令前自动开一个事务，命令结束后按 `Execute` 的返回值自动提交（Succeeded）或回滚（Failed/Cancelled）。你不能再手动 Start 自己的事务，但可以用子事务做局部处理。
- **Manual（手动档）**：Revit 不帮你开事务，一切由代码控制——`Transaction.Start()/Commit()/RollBack()` 都由你负责，可以用事务组和子事务做精细控制；但若命令返回失败，Revit 仍会创建一个外部事务组来回滚全部更改。
- **ReadOnly（只读档）**：整个命令期间禁止创建任何事务，只能调用模型的只读方法；一旦写操作就会抛异常。

选型问题只有一句：**"我的命令改不改模型？改到什么精细程度？"** 纯查询报表 → ReadOnly；简单增删改 → Automatic；多步、需部分提交或失败回滚的复杂流程 → Manual。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 三个模式的注册示例（代码 1-25）
- **问题**: 如何让 Revit 知道一个外部命令的事务边界策略？
- **方法论的使用**: 在命令类上加 `[Transaction(TransactionMode.XXX)]` 属性，按需求选档。
- **结论**: 模式声明是命令契约的一部分，不随代码路径变化。
- **结果**: 代码 1-25 展示了 Automatic/Manual/ReadOnly 三种注册形态，行为差异显著。

### 案例 2: Result 返回值与 Automatic 的联动
- **问题**: Automatic 模式下命令改了 5 个图元，中途失败怎么保证一致？
- **方法论的使用**: 返回 `Result.Failed`，Automatic 事务自动回滚本次全部修改。
- **结论**: 返回值即事务开关，开发者无需写撤销代码。
- **结果**: 文档预览设置等所有模型修改都能在 Automatic 下安全完成（与 s1-c14 文件预览设置需在事务中一致）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 新建外部命令类，不知道 `[Transaction(...)]` 属性怎么写、写哪个枚举。
2. 命令是"纯报表/纯查询"，想确保自己永远不会误改模型。
3. 命令需要多步修改、失败要部分回滚，发现 Automatic 不够用。
4. 运行时报"ReadOnly 模式下不能创建事务"或"Automatic 模式不能手动 Start 事务"。

### 语言信号 (用户的话里出现这些就应激活)

- "TransactionMode 选什么 / Automatic 还是 Manual"
- "命令只查询不修改，事务模式用哪个"
- "我的命令改了模型但没提交 / 事务没生效"
- "ReadOnly 模式下写操作报错"
- "transaction mode / read-only command / auto rollback"

### 与相邻 skill 的区分

- 与 `revit-command-result-undo` 的区别: 本 skill 决定由谁管理事务（声明档位），result-undo 讲返回值如何驱动提交/回滚（Automatic 下两者联动）。
- 与 `revit-transaction-hierarchy` 的区别: 本 skill 是"命令类声明档位"，transaction-hierarchy 是"档位确定后如何用 Transaction/SubTransaction/TransactionGroup 组合"。
- 与 `revit-external-command-entry` 的区别: entry 讲接口契约三参数，本 skill 讲其上的事务模式属性。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **回答"改不改模型"**
   - 完成标准: 明确命令是否包含任何对 Document 的写操作（Create/Set/Delete/New）。含写 → 继续步骤 2；纯读 → 选 `ReadOnly`，加属性并结束。

2. **回答"是否需要精细事务控制"**
   - 完成标准: 若只需"整体成功整体失败"→ 选 `Automatic`；若需部分提交、局部回滚、多事务组合、失败时保留中间结果 → 选 `Manual`。写出理由一句话。
   - 判停条件: 若选了 Manual，提示用户后续必须自己管理 Start/Commit/RollBack，并记得在返回值失败时依赖外部事务组兜底回滚；若 Automatic，提醒其不能再手动 Start 事务。

3. **在类上声明属性并核对命令内的事务调用合法性**
   - 完成标准: `[Transaction(TransactionMode.XXX)]` 已加在类上；Automatic 命令内无手动 `Transaction.Start()`；ReadOnly 命令内无任何写 API。检查完列出该模式下"允许/禁止"清单给用户。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 事务模式已定、正在写事务内部逻辑（Start/Commit/SubTransaction）——交给批次2事务 skill。
- 非命令场景（如 Updater、事件回调）——它们的事务约束不同，不能套用 `TransactionMode`。

### 作者在书中警告的失败模式

- Automatic 模式下仍尝试 `new Transaction().Start()` → 抛异常，因为 Revit 已替你创建事务。
- ReadOnly 模式下任何写操作直接抛异常，误以为"只是不提交"是错的。
- Manual 模式下忘记 Commit 且返回 Succeeded → 修改丢失；返回 Failed 才由外部事务组回滚。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：后续版本对 `TransactionMode` 行为有微调（如 Automatic 下 Regenerate 时机的差异），决策树仍适用但异常信息可能不同。
- 书中未强调：ReadOnly 并非"零开销"，只读命令仍可能触发图元惰性加载，性能问题不归本 skill 管。

### 容易混淆的邻近方法论

- "Automatic 的自动回滚"≠"手动 RollBack"——前者只在返回 Failed/Cancelled 时发生，后者可随时按需触发。
- `TransactionMode`（类属性，声明式）与 `Transaction`（类，命令式）是两个层次：前者是策略，后者是执行工具。

---

## 相关 skills

- revit-command-result-undo：composes-with——本 skill 声明事务档位，result-undo 讲返回值如何驱动提交/回滚，Automatic 模式下两者联动。
- revit-transaction-hierarchy：composes-with——本 skill 选档位，transaction-hierarchy 讲档位确定后 Transaction/SubTransaction/TransactionGroup 的分层组合。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
