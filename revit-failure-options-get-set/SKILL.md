---
name: revit-failure-options-get-set
description: |
  配置某个事务的故障处理时使用。核心：FailureHandlingOptions 无法 new，必须 Get→修改→Set 写回。何时调用：按事务定制故障处理、挂 IFailuresPreprocessor、关弹窗。何时不调用：只读配置。Trigger：'配置失败处理/挂预处理器/SetFailuresPreprocessor'（failure handling options, set failures preprocessor）。代码里写 new FailureHandlingOptions() 即为错误写法。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2.3（约p330）
tags: [failure-handling, options, transaction, preprocessor, api-pattern]
related_skills:
  - slug: revit-transaction-hierarchy
    relation: composes-with
  - slug: revit-failure-handling-options
    relation: composes-with
---

# SetFailureHandlingOptions 无法 new，必须 Get + Set 配套使用

## R — 原文 (Reading)

> SetFailureHandlingOptions() 方法采用 FailureHandlingOptions 对象作为参数。无法创建此对象，必须使用 GetFailureHandlingOptions() 方法从事务中获得。……
>
> — 宦国胜, 第5章 5.2.3（约p330）

---

## I — 方法论骨架 (Interpretation)

Revit 里有一个违反直觉的 API 约束：`FailureHandlingOptions`（故障处理选项）这个对象**没有公开构造函数，你不能 new 一个**。它的实例只能从某个事务里 `GetFailureHandlingOptions()` 取出来，修改后再 `SetFailureHandlingOptions()` 写回同一个事务。

原因是它的所有权归 Revit 内部：选项对象与**特定事务的状态绑定**，携带了该事务的故障事件订阅者等信息。你把配置"借出来"改一改再"还回去"，Revit 才能保证配置和事务生命周期一致。

同样约束适用于整条注册链：要挂 `IFailuresPreprocessor`，也是先 `GetFailureHandlingOptions()` → `SetFailuresPreprocessor(new MyPre())` → `SetFailureHandlingOptions(opts)` 三连。别想着一份 options 实例跨多个事务复用——每个事务都要 Get+Set 一套。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 尝试共享一份配置对象
- **问题**: V2 预测场景——两个事务想共享同一份 FailureHandlingOptions 配置对象，能否 new 一个实例反复用？
- **方法论的使用**: 检查 API 契约——构造函数非 public，且 options 与事务状态绑定。
- **结论**: 不能。Revit 内部维护其生命周期，new 会在编译期就失败。
- **结果**: 必须对每个事务单独 GetFailureHandlingOptions() 取副本、修改、Set 写回。

### 案例 2: 挂事务级故障预处理器
- **问题**: 第5章 5.8.2 里 IFailuresPreprocessor 要接入某个事务。
- **方法论的使用**: 走 GetFailureHandlingOptions 链：Get → SetFailuresPreprocessor → SetFailureHandlingOptions 写回。
- **结论**: 预处理器注册必须经 options 对象间接完成，不能直接 new 挂载。
- **结果**: 该事务独享自定义预处理逻辑，其他事务不受影响（对应 seg4b-f11）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 想对某个事务"关掉警告弹窗"或"回滚后清空警告"——需要改它的 FailureHandlingOptions。
2. 给某个事务挂自定义 IFailuresPreprocessor 做事务级压噪。
3. 代码里写了 `new FailureHandlingOptions()`，编译不过或行为诡异。
4. 想让批量事务的警告延迟到批次末统一处理（DelayedMiniWarnings）。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么关掉这个事务的警告弹窗？" / "suppress warning dialog for this transaction"
- "FailureHandlingOptions 怎么 new？" / "create FailureHandlingOptions instance"
- "给事务挂个故障预处理器" / "attach a failures preprocessor to my transaction"
- "回滚后警告还在，怎么清空？" / "clear warnings after rollback"
- "配置对象能不能复用？" / "reuse the same failure options"

### 与相邻 skill 的区分

- 与 `revit-transaction-hierarchy` 的区别: 本 skill 讲事务的故障处理配置（Get+Set 链），transaction-hierarchy 讲事务层级结构本身；配置挂在具体事务上，先有事务再谈配置。
- 与 `revit-failures-preprocessor`（revit-failures-preprocessor）：那个讲 IFailuresPreprocessor 接口本身的行为；本 skill 讲注册它的手筋（Get+Set 链）。
- 与 `revit-exception-types`（revit-exception-types）：那是异常类型知识；本 skill 是故障处理选项的配置手筋，不冲突。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **从事务取 options 副本**：确认事务已 Start，`var opts = transaction.GetFailureHandlingOptions();`。
   - 完成标准: 拿到 options 实例（不是 new 的，也不是 null）。
   - 判停条件: 若代码里出现 `new FailureHandlingOptions()`，**停下**，删除，改为 Get。

2. **用 Set 方法配置**：按需调用 `SetClearAfterRollback(bool)` / `SetDelayedMiniWarnings(bool)` / `SetForcedModalHandling(bool)` / `SetFailuresPreprocessor(IFailuresPreprocessor)` / `SetTransactionFinalizer(...)`。
   - 完成标准: 所需选项已逐一 Set 上，参数值明确。

3. **写回并验证**：`transaction.SetFailureHandlingOptions(opts);`。
   - 完成标准: 写回调用成功；提交/回滚时观察到对应行为（如弹窗消失、警告被清）。
   - 注意事项: 每个事务都要独立走一遍 Get→Set 链，不要跨事务共享同一实例。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只想读配置、不修改——直接 Get 看完即可，不必写回。
- 没有开启事务——事务外 GetFailureHandlingOptions 无意义/会抛异常。
- 想全局改变所有事务的默认故障处理——那是 IFailuresProcessor（全局）的职责，不是每个事务配 options。

### 作者在书中警告的失败模式

- **无法创建此对象**——new 直接编译错误，这是硬性 API 约束，不是"建议"。
- **Get 后不 Set 写回**——修改丢失，配置不生效，且可能造成"读的是一份、事务用的是另一份"的错觉。
- **跨事务复用同一 options 实例**——对象与特定事务状态绑定，复用违背所有权模型，行为未定义。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 API：新版中部分 FailureHandlingOptions 的 setter 签名与默认值可能有调整，应以当前 SDK 文档为准。
- 未展开说明"options 与事务绑定"的内部实现（订阅者列表随事务销毁），理解所有权模型即可，不必依赖内部细节。

### 容易混淆的邻近方法论

- `FailureHandlingOptions`（事务级，Get+Set 写回） vs `IFailuresProcessor`（全局，注册即接管所有会话错误）：作用域不同，别混用。
- "Set 方法" 与"修改属性"：FailureHandlingOptions 用 Set 方法族配置，而不是直接改公共字段/属性——注意 API 形态。

---

## 相关 skills

- revit-transaction-hierarchy：composes-with——本 skill 讲在具体事务上挂故障处理配置，transaction-hierarchy 讲事务/子事务/事务组的层级结构，二者共同构成事务的完整用法。
- revit-failure-handling-options：composes-with——本 skill 是"Get+Set 配套"的获取方式，failure-handling-options 是五个 Set 方法的语义，合起来是事务故障处理的完整配置面。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
