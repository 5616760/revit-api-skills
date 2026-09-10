---
name: revit-enableworksharing-irreversible
description: |
  调用 Document.EnableWorksharing() 前的必读反例：该操作清除文件的撤销历史记录——命令本身及
  之前的所有操作都无法撤销；调用前必须让所有显式启动的事务阶段（事务/子事务/事务组）全部完成，
  否则失败。正确时机：所有事务 Commit 后、不在任何 TransactionGroup 内调用。Trigger：
  "启用工作共享"、"EnableWorksharing"、"能不能撤销"、"事务组内启用共享"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.13.6 启用工作共享（约p405）
tags: [worksharing, irreversible, counter-example, transactions, revit-api]
related_skills: []
---

# EnableWorksharing 不可撤销

## R — 原文 (Reading)

> "通过 Revit.API 使用 Document.EnableWorksharing(worksharing) 方法来启用工作共享。此命令将清除文件的撤销历史记录，因此该命令以及在此之前执行的其他操作无法撤销。"
>
> — 宦国胜，第5章 5.13.6 启用工作共享 约p405

---

## I — 方法论骨架 (Interpretation)

EnableWorksharing 是一次"单向转换"操作——把普通文档变成工作共享文档，代价是摧毁撤销栈。

- **不可逆点一：清除撤销历史**。调用后，该命令本身及之前的一切操作都无法撤销。更隐蔽的推论：包在外层的 `TransactionGroup` 的 Assimilate/RollBack 也将失效——因为撤销栈已被清空，事务组的回滚承诺作废。
- **不可逆点二：前置完成条件**。调用时若有任何活跃的 Transaction / SubTransaction / TransactionGroup 未提交/未结束，操作会失败。
- 正确调用时机（安全窗口）：
  1. 所有事务已 Commit；
  2. 不在任何 TransactionGroup 的 Start/RollBack 区间内；
  3. 用户已被告知"启用后不能撤销之前的操作"。
- 检查清单式记忆：调用前问三个问题——"撤销栈可以丢弃吗？事务都关了吗？不在事务组里吗？"
- 与一般软件对比："启用某功能"通常是无害开关；这里是破坏性结构转换——单向门。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 在 TransactionGroup 内调用 EnableWorksharing

- **问题**: 开发者想用 TransactionGroup 包住"启用工作共享 + 一系列初始化操作"，失败时整体回滚。
- **方法论的使用**: 按"清除撤销历史"推导——事务组的 RollBack/Assimilate 依赖撤销栈，栈被清空后失效；且活跃事务组未结束时调用直接失败。
- **结论**: 不存在"可回滚的启用"，安全窗口在事务组 Start 之前、所有事务 Commit 之后。
- **结果**: 调整调用顺序后，启用成功且失败路径清晰。

### 案例 2: 启用前有未提交事务

- **问题**: 事务未 Commit 就调用 EnableWorksharing，操作失败。
- **方法论的使用**: 书中明示"显式启动的所有事务阶段务必完成"。
- **结论**: 完成所有事务阶段是硬前置条件。
- **结果**: 补 Commit 后调用成功。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要为单机文档开启协作（创建中心文件流程的第一步）。
2. 代码在事务/事务组内调用 EnableWorksharing 失败。
3. 评审"启用后还能不能撤销"类的流程安全问题。

### 语言信号 (用户的话里出现这些就应激活)

- "EnableWorksharing / 启用工作共享"（enable worksharing）
- "启用后能不能撤销"（can I undo after enabling）
- "事务组内调用失败 / TransactionGroup 内启用"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **检查事务状态**
   - 确认当前无活跃 Transaction/SubTransaction/TransactionGroup（可通过代码路径审查确认）。
   - 完成标准: 全部事务阶段已 Commit/结束，无未闭合调用。
   - 判停条件: 存在活跃事务 → 先完成或回滚它们，否则调用必然失败，到此停止。

2. **评估不可逆影响并告知**
   - 向用户/调用方确认：启用后之前的操作不可撤销。
   - 完成标准: 有显式的告知或确认步骤（UI 提示/日志记录）。

3. **在安全窗口调用**
   - 所有事务 Commit 后、任何事务组 Start 之前调用 EnableWorksharing；随后再开新事务做工作集划分等后续操作。
   - 完成标准: 调用成功，后续操作在全新撤销历史中进行。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 文档已启用工作共享（IsWorkshared == true）——无需也无法再次启用。
- 想在"试用后回滚"的实验流程里启用——不存在该路径，启用即永久。

### 作者在书中警告的失败模式

- 活跃事务未完成时调用 → 失败（本单元本体）。
- 以为 TransactionGroup 能兜底回滚 → 撤销栈被清，回滚承诺作废。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版中启用工作共享的对话框与工作集默认命名有变化，但"清撤销栈、事务前置完成"两条硬约束长期有效。

### 容易混淆的邻近方法论

- 事务三层级（Transaction/SubTransaction/TransactionGroup）：本 skill 的前置条件正是"三层级全部收尾"——层级语义是其基础。
- 保存类操作（SaveAs）也是部分不可逆节点，但不破坏撤销栈，程度不同。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
