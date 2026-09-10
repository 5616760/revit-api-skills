---
name: revit-failures-accessor
description: |
  在故障处理三步中读故障、删警告、解决、删图元、替换故障时使用，FailuresAccessor 是公共入参。方法族：GetFailuresMessages、DeleteWarning(s)、ResolveFailure(s)、ReplaceFailures，配 FailureProcessingResult 结果码。何时调用：自动故障处理。Trigger：'读取故障/删警告/ResolveFailure/FailureProcessingResult 选哪个'（failures accessor）。陷阱：已回滚后无法返回 Commit；无解决方案的 Commit 无限循环。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.8.2（约p371-372）
tags: [failures-accessor, failure-resolution, warning, processing-result, api]
related_skills:
  - slug: revit-failures-processing-steps
    relation: depends-on
  - slug: revit-failures-preprocessor
    relation: composes-with
---

# FailuresAccessor 解决选项与结果控制

## R — 原文 (Reading)

> FailuresAccessor 对象作为参数传递到故障处理的每个步骤……通过 GetFailuresMessages()方法……DeleteWarning()或 DeleteAllWarnings()……ResolveFailure()或 ResolveFailures()……
>
> — 宦国胜, 第5章 5.8.2 故障访问器（约p371-372）

---

## I — 方法论骨架 (Interpretation)

`FailuresAccessor` 是故障处理三步流水线（Preprocessor → 事件 → Processor）的**公共入参**——无论你在哪一步，都通过它访问/操纵"当前事务的故障集合"。方法族分四类：

- **读取**：`GetFailuresMessages()` 拿全部故障消息，可遍历判断类型与严重程度。
- **删除**：`DeleteWarning()` / `DeleteAllWarnings()`——把已知噪音警告直接清掉（最常用于压噪）。
- **解决**：`ResolveFailure()` / `ResolveFailures()`——标记某个故障"已解决"（若故障带 FailureResolution）。
- **结构性操作**：`DeleteElements()`（删掉出问题的图元来"解决"故障）、`ReplaceFailures()`（把一堆噪音故障压缩合并成一个通用故障）。

再配合 **`FailureProcessingResult`** 结果码决定引擎下一步：
- `ProceedWithCommit`（继续提交）/ `ProceedWithRollBack`（回滚）/ 其他（如 Continue）。

两个必须记住的语义陷阱：
1. **事务已回滚后无法返回 Commit**——如果你请求 ProceedWithCommit 但事务已经回滚了，它会被当作 ProceedWithRollBack 处理。
2. **ProceedWithCommit 没有实际解决方案 → 无限循环**——引擎发现故障没解决会再触发一轮处理；必须保证"每个 Commit 都有对应的真实解决动作"。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 回滚后请求继续提交
- **问题**: V2 预测场景——事务已回滚后还想返回 ProceedWithCommit 继续提交，会怎样？
- **方法论的使用**: 检查结果码语义——"若已回滚则无法返回 Commit"。
- **结论**: 被视为 ProceedWithRollBack，结果码必须与当前状态一致。
- **结果**: 避免写出"死而复活"的提交逻辑。

### 案例 2: 噪音故障压缩
- **问题**: 一个事务触发了几十条相似警告，用户弹窗刷屏。
- **方法论的使用**: ReplaceFailures() 把一堆噪音故障压缩成一个通用故障。
- **结论**: 用一条消息替代几十条。
- **结果**: 用户看到一条简洁提示，而不是刷屏。

### 案例 3: 删图元解决故障
- **问题**: 某些故障的本质是"图元本身不合法"。
- **方法论的使用**: DeleteElements() 用删除相关图元的方式解决故障。
- **结论**: 删除后故障自动消失，事务可收敛提交。
- **结果**: 自动化处理路径覆盖"删除型"故障。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 自动处理故障时，需要遍历故障消息判断"哪些能删、哪些要解决"。
2. 几十条相似警告刷屏，要压成一条。
3. 故障的本质是图元非法，要用删除来解决。
4. 选择 FailureProcessingResult 时不确定"回滚后能不能 Commit"。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么读取当前事务的故障？" / "get failures messages from the transaction"
- "把警告都删掉 / DeleteAllWarnings" / "delete all warnings"
- "ResolveFailure 怎么用？" / "how to resolve a failure"
- "ReplaceFailures 压缩警告" / "collapse many failures into one"
- "回滚后还能提交吗？" / "can I commit after rollback"

### 与相邻 skill 的区分

本 skill 与 `revit-failures-processing-steps` 区分：流水线回答"在哪几步处理"，本 skill 回答"每一步里用什么工具（accessor 方法族）处理"；`revit-failures-preprocessor` 拿到的公共入参正是 FailuresAccessor，本 skill 是预处理器/事件/处理器里可调用的操作集合。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **遍历并分类故障**：在任一步骤里用 `accessor.GetFailuresMessages()` 拿到全部故障，按类型/严重程度分类。
   - 完成标准: 明确"哪些可安全删除、哪些需要解决、哪些要删除图元、哪些要保留让用户看"。

2. **选择处置手段**：
   - 已知噪音 → DeleteWarning()/DeleteAllWarnings()。
   - 有标准解决动作 → ResolveFailure()。
   - 图元非法 → DeleteElements()。
   - 故障过多 → ReplaceFailures() 合并。
   - 完成标准: 每个分类都有明确处置手段。

3. **返回收敛的结果码**：
   - 确认处置后故障确实消失，才返回 ProceedWithCommit；否则返回 ProceedWithRollBack。
   - 检查事务当前状态：已回滚就不要再请求 Commit。
   - 完成标准: 结果码与事实一致；处理链收敛（无无限循环、无静默失败）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 不干预故障处理（接受默认交互）——不需要 accessor。
- 只是读取故障展示——GetFailuresMessages 够用，别做多余处置。

### 作者在书中警告的失败模式

- **回滚后请求 Commit → 被降级为 RollBack**——结果码与状态不一致。
- **ProceedWithCommit 无解决方案 → 无限循环**——引擎会反复触发处理。
- **DeleteAllWarnings 可能误删重要警告**——要判断清楚哪些是噪音，别一刀切。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 FailuresAccessor 增加了一些方法（如 GetFailuresToProcess 相关 API 演进），应按当前 SDK 核对方法名。
- 未讨论"处理器内删除图元是否再触发新故障"的连锁反应——自动化删除需实测。

### 容易混淆的邻近方法论

- `DeleteWarning`（清除警告，事务可继续） vs `ResolveFailure`（标记已解决，需 FailureResolution）——前者"抹掉"，后者"解决"。
- `FailureProcessingResult` 各值（ProceedWithCommit / ProceedWithRollBack / Continue）不是随便选的——每个都绑定事务下一步实际动作。

---

## 相关 skills

- revit-failures-processing-steps（depends-on）：流水线定义"在哪几步处理"，本 skill 提供每一步里用的 accessor 方法族。
- revit-failures-preprocessor（composes-with）：预处理器拿到的公共入参正是 FailuresAccessor，本 skill 是其中可调用的操作集合。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
