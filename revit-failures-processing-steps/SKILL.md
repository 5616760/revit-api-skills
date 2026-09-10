---
name: revit-failures-processing-steps
description: |
  理解/干预事务提交时故障处理流水线时使用。三步：IFailuresPreprocessor → FailuresProcessing 事件 → IFailuresProcessor，每步用 FailureProcessingResult 控制下一步，可能多次循环。何时调用：自动故障处理、排查事务无声中止。何时不调用：接受默认弹窗。Trigger：'自动处理故障/预处理器/事件处理器/FailureProcessingResult'（failures processing pipeline, silently rolled back）。危险：全局处理器传 null = 全部静默回滚。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.2（约p370-371）
tags: [failures-processing, preprocessor, processor, event, pipeline]
related_skills: []
---

# 故障处理三步骤框架（Preprocessor → FailuresProcessing → Processor）

## R — 原文 (Reading)

> 故障处理的每个循环包括三个步骤：(1)故障预处理（IFailuresPreprocessor）。(2)故障处理事件广播（FailuresProcessing 事件）。(3)最终处理（IFailuresProcessor）。……
>
> — 宦国胜, 第5章 5.8.2（约p370-371）

---

## I — 方法论骨架 (Interpretation)

事务提交时如果带故障，Revit 不会直接给结果，而是走一条**三步流水线**，每步都有机会"把问题解决掉"：

1. **IFailuresPreprocessor（事务级预处理）**：只能为特定事务注册（每个事务最多一个、无默认）。在故障解决过程里**最先**拿到控制权，适合"这个事务的已知噪音我压掉"。
2. **FailuresProcessing 事件（广播）**：预处理器完成后触发，可有任意数量处理程序，适合无 UI 的自动处理。
3. **IFailuresProcessor（全局最终处理）**：全局唯一的处理器，替代标准 Revit 错误对话框，负责兜底。

每一步通过 `FailureProcessingResult` 告诉引擎下一步怎么走：
- `Continue` / `ProceedWithCommit` / `ProceedWithRollBack` / `ProceedWithCommitWithOptions`……**返回值控制后续**。比如预处理器没解决 → 事件处理器被调；事件处理器也没解决 → 全局处理器兜底。
- 关键：**Commit 与实际处理之间可能发生多次循环**——如果某步返回"继续提交"但故障还在，引擎会再触发一轮处理；没有真实解决方案的 ProceedWithCommit 会导致无限循环（见 revit-failures-accessor/f12）。

这个流水线的意义：插件的故障处理可以完全**无 UI、无弹窗、自动化**——把预处理器/事件/处理器依次排好，故障就在提交时被静默消化。但危险性也在"静默"：如果全局处理器传了 null，所有故障事务将**无声中止（静默回滚）**，用户根本不知道为什么没提交成功。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 全局处理器传 null 的后果
- **问题**: V2 预测场景——注册一个 IFailuresProcessor 但传了 null，之后所有带故障的事务提交会怎样？
- **方法论的使用**: 按三步流水线推演——最终处理环节没有实际处理器。
- **结论**: 所有故障事务将无声中止（除非前两步已解决故障）——传 null 相当于"全部静默回滚"。
- **结果**: 极危险的全局配置，排查困难（没有任何提示）。

### 案例 2: 三步各司其职
- **问题**: 想实现完全自动的故障处理，从哪里下手？
- **方法论的使用**: 按流水线分工——Preprocessor 删警告/解决/中止 → 事件广播自动处理 → Processor 兜底替代标准 UI。
- **结论**: 三步顺序执行，每步可控制下一步。
- **结果**: 自动故障处理体系的骨架清晰化。

### 案例 3: 多次处理循环
- **问题**: 事务提交后的处理是一次性的吗？
- **方法论的使用**: 认识到 Commit 与实际处理间可能发生多次循环。
- **结论**: 处理结果决定是否再来一轮。
- **结果**: 设计处理器时必须保证"最终会收敛"（要么解决、要么回滚），否则死循环。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要写自动故障处理（无弹窗批处理）——需要确定用哪一步、怎么组合。
2. 排查"事务无声中止/莫名其妙回滚"——怀疑全局处理器配置问题。
3. 想控制"预处理器解决不了就抛给事件，还不行就全局兜底"的分级策略。
4. 理解 FailureProcessingResult 各值对下一步的影响。

### 语言信号 (用户的话里出现这些就应激活)

- "自动处理事务里的警告" / "automatically handle failures in transactions"
- "故障预处理器/事件/处理器怎么分工？" / "preprocessor vs event vs processor"
- "事务没报错但没提交成功？" / "transaction silently rolled back"
- "FailureProcessingResult 每个值什么意思？" / "meaning of each FailureProcessingResult"
- "怎么让批处理不弹任何对话框？" / "no dialogs at all during batch"

### 与相邻 skill 的区分

本 skill 是故障处理三步流水线的总览与顺序框架；`revit-failures-preprocessor`、`revit-failures-processing-event`、`revit-failures-processor-global` 分别是每步的细节实现。与 `revit-failure-handling-options` 区分：options 决定"弹不弹窗、挂不挂钩子"，流水线决定"故障实际怎么被处理"；与 `revit-failure-definition-registration` 区分：那是生产故障（发布），本 skill 是消费故障（处理）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定处理需求与分工**：列出要处理的故障类型；决定各用哪一步——事务级噪音用 Preprocessor；全局自动处理用事件/Processor；需要替代标准 UI 才用全局 Processor。
   - 完成标准: 三步中明确用哪几步、各自职责，避免重复处理同一故障。
   - 判停条件: 若只是想临时压掉一次批处理的警告，优先事务级 Preprocessor，别升级到全局 Processor。

2. **实现并注册**：
   - Preprocessor：GetFailureHandlingOptions → SetFailuresPreprocessor → Set 写回（事务级）。
   - 事件：订阅 Application.FailuresProcessing，用 args.SetProcessingResult 传结果。
   - Processor：通过 API 注册全局处理器（**不要传 null**）。
   - 完成标准: 注册路径正确、作用域正确（事务级 vs 全局）。

3. **验证收敛性**：
   - 构造带故障的事务跑一遍，确认：处理链能收敛（最终 Commit 或 RollBack，不会无限循环）。
   - 检查每个 ProceedWithCommit 都对应真实解决方案。
   - 完成标准: 故障被正确处理（解决/回滚）；无静默中止、无死循环。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 接受默认弹窗交互——不需要流水线干预。
- 只读操作（无事务提交）——没有故障处理发生。
- 只想压一次特定警告——事务级 Preprocessor 足够，别全局接管。

### 作者在书中警告的失败模式

- **全局处理器传 null → 所有故障事务无声中止**——最难排查的坑。
- **ProceedWithCommit 无真实解决方案 → 无限循环**——设计每一步都要能收敛。
- **提交后事务已回滚，还想返回 Commit** → 被降级为 RollBack（revit-failures-accessor）。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 FailureProcessingResult 枚举值、以及故障引擎的循环次数上限有调整，应以当前 SDK 为准。
- 未讨论"处理器中能否再次修改模型"等细粒度限制——实现时需实测。

### 容易混淆的邻近方法论

- `IFailuresPreprocessor`（事务级、先执行、每事务最多一个） vs `IFailuresProcessor`（全局、最后执行、唯一）：作用域与时机相反。
- 故障处理流水线 vs .NET 异常：前者在事务收尾裁决"提交还是回滚"；后者在代码层抛出/捕获——两套体系，不要互相替代。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
