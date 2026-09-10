---
name: revit-failures-preprocessor
description: |
  给特定事务做故障预处理（压噪）时使用。规则：每事务最多一个预处理器、无默认、故障解决过程最先获控制；注册经 GetFailureHandlingOptions→SetFailuresPreprocessor→Set 写回。何时调用：批处理压掉已知噪音警告（如 RoomNotEnclosed）、不影响全局配置。何时不调用：需要全局生效。Trigger：'只压这个事务的警告/事务级故障处理'（failures preprocessor, transaction-scoped suppress）。作用域仅限该事务。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.2（约p372-373）
tags: [failures-preprocessor, transaction-scope, suppress-warnings, custom-handling]
related_skills:
  - slug: revit-failures-processing-steps
    relation: depends-on
  - slug: revit-failure-options-get-set
    relation: depends-on
  - slug: revit-failures-processor-global
    relation: contrasts-with
  - slug: revit-failures-processing-event
    relation: contrasts-with
---

# IFailuresPreprocessor 事务级预处理

## R — 原文 (Reading)

> IFailuresPreprocessor 仅可用于为特定事务提供自定义故障处理……在故障解决过程期间首先获取控制……每个事务可能只有一个 IFailuresPreprocessor，且没有默认的故障预处理器。
>
> — 宦国胜, 第5章 5.8.2 故障预处理器接口（约p372-373）

---

## I — 方法论骨架 (Interpretation)

`IFailuresPreprocessor` 是故障流水线的**第一棒**，特点是"小而专"：

- **作用域**：只能为**特定事务**提供故障处理——给这个事务挂上，就只管这个事务的故障，其他事务、其他会话一概不受影响。
- **数量**：每个事务最多挂一个；没有默认预处理器（不挂就没有）。
- **时机**：故障解决过程中**最先**获得控制权——你有第一机会把故障"解决掉"或"压掉"，处理不了的再往下游抛（事件、全局处理器）。
- **注册方式**：`GetFailureHandlingOptions()` → `SetFailuresPreprocessor(new MyPre())` → `SetFailureHandlingOptions(opts)` 三连（配置对象必须 Get+Set 写回，见 revit-failure-options-get-set）。

典型用途是"事务级压噪"：批量脚本里每个事务都会触发 `RoomNotEnclosed` 之类已知噪音警告，你只在这一批事务里挂个预处理器把它 `DeleteWarning` 掉——精准、隔离、不动用户全局配置。相比在 FailuresProcessing 事件里做全局过滤，它的影响面小得多、更安全。

预处理器方法 `PreprocessFailures(FailuresAccessor accessor)` 里可以：遍历故障、删警告、解决、决定结果——返回值是 `FailureProcessingResult`，控制流水线下一步。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批处理压掉 RoomNotEnclosed 刷屏
- **问题**: V2 预测场景——批量脚本里每个事务都触发 RoomNotEnclosed 警告刷屏，想只在这批脚本里压掉、不影响用户全局配置。
- **方法论的使用**: 用事务级 IFailuresPreprocessor，在 PreprocessFailures 里 DeleteWarning 已知噪音。
- **结论**: 作用域仅限该事务（每个事务最多一个、无默认、不影响其他会话）。
- **结果**: 比全局事件过滤更精准，比弹窗交互更可靠。

### 案例 2: 注册链
- **问题**: 怎么把预处理器挂上去？
- **方法论的使用**: 第5章 5.8.2 + 5.2.3——GetFailureHandlingOptions → SetFailuresPreprocessor(new MyPreprocessor()) → SetFailureHandlingOptions。
- **结论**: 注册必须经 options 对象间接完成。
- **结果**: 预处理器对该事务生效，事务结束自动解除。

### 案例 3: 与其他步骤的衔接
- **问题**: 预处理器解决不了的故障怎么办？
- **方法论的使用**: 返回结果让流水线继续——处理不了就放给 FailuresProcessing 事件和全局 Processor。
- **结论**: 预处理器是"第一机会"，不是"唯一机会"。
- **结果**: 分级处理策略成型（revit-failures-processing-steps 三步框架的落地）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批量脚本有已知噪音警告要压掉，且不想动全局配置。
2. 只想给某个特殊事务做定制故障处理，其他事务照旧。
3. 问"预处理器和全局处理器有什么区别、什么时候用哪个"。
4. 故障处理想从"第一步"介入而不是等全局兜底。

### 语言信号 (用户的话里出现这些就应激活)

- "只压掉这个事务的警告" / "suppress warnings for this transaction only"
- "预处理器和处理器有什么区别？" / "preprocessor vs processor"
- "IFailuresPreprocessor 怎么注册？" / "how to register an IFailuresPreprocessor"
- "RoomNotEnclosed 这类已知警告怎么屏蔽？" / "filter known warning noise"
- "事务级故障处理" / "transaction-scoped failure handling"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-processor-global` 区分：预处理器是事务级、最先、每事务最多一个；全局处理器是全会话级、最后、唯一并替代标准 UI。与 `revit-failures-processing-event` 区分：事件在预处理器之后触发、可有任意多个处理程序。注册预处理器必须经 `revit-failure-options-get-set` 的 Get+Set 链，整体框架见 `revit-failures-processing-steps`。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **实现 IFailuresPreprocessor**：新建类实现 `PreprocessFailures(FailuresAccessor accessor)`，在其中识别并处置已知噪音（DeleteWarning/ResolveFailure），返回 FailureProcessingResult。
   - 完成标准: 类已实现；对"想压的警告"有明确匹配条件；返回值明确（解决→Commit，未解决→Continue/给下游）。

2. **注册到目标事务**：
   - 事务 Start 后：`var opts = transaction.GetFailureHandlingOptions(); opts.SetFailuresPreprocessor(new MyPre()); transaction.SetFailureHandlingOptions(opts);`
   - 完成标准: 注册成功，该事务独享预处理。

3. **验证作用域与效果**：
   - 跑目标事务：噪音警告被压掉，流程收敛。
   - 跑一个**未注册**该预处理器的其他事务：确认不受影响。
   - 完成标准: 影响范围精确限定在该事务；其他事务行为不变。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要全局生效的故障处理——用事件或全局 Processor（revit-failures-processing-event/f13），别给每个事务重复挂。
- 事务本身不会产生故障——没必要挂。
- 想替换标准错误对话框——那是全局 Processor 的职责，预处理器不做 UI 接管。

### 作者在书中警告的失败模式

- **每个事务最多一个预处理器**——挂第二个会失败/覆盖，别试图叠加。
- **无默认预处理器**——不挂就没有，别假设系统自带。
- **预处理器处理不了却返回"已解决"**——说谎式返回会导致事务带着故障提交，风险转移而非消除。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版预处理器接口方法签名（如是否带 document 上下文）可能有调整，应按当前 SDK 实现。
- 未讨论"一个插件在多个文档上、每个文档都要挂预处理器"的繁琐——工程上要写辅助函数批量挂载。

### 容易混淆的邻近方法论

- `IFailuresPreprocessor`（事务级、最先、唯一） vs `IFailuresProcessor`（全局、最后、唯一）：名字只差一个 Pre，作用域差一个量级。
- "事务级压噪" vs "全局过滤警告"：前者隔离、安全；后者省事但影响全会话——优先级选前者。

---

## 相关 skills

- revit-failures-processing-steps（depends-on）：本 skill 是三步流水线的第一棒，总览框架见该 skill。
- revit-failure-options-get-set（depends-on）：注册预处理器必须走 Get+Set 链，本 skill 讲接口行为。
- revit-failures-processor-global（contrasts-with）：事务级最先 vs 全会话级最后，作用域与时机相反。
- revit-failures-processing-event（contrasts-with）：预处理器唯一且最先，事件可多个且在其后触发。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
