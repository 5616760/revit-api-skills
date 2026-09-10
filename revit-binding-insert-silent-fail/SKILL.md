---
name: revit-binding-insert-silent-fail
description: |
  当共享参数绑定操作可能失败且需要防御性检查时调用。
  不适用于：首次绑定且确定无冲突、可扩展存储。
  关键 trigger 信号："Insert 返回 false insert returns false"、"绑定失败 binding failed"、"静默失败 silent failure"、"同时绑 instance 和 type"、"重复绑定 duplicate binding"。
  核心认知：BindingMap.Insert 不抛异常，第二次绑定（Type↔Instance 冲突）静默返回 false，必须检查返回值。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.3 (约p319)
tags: [binding, silent-failure, insert-returns-false, shared-parameter, revit-api]
related_skills:
  - slug: revit-schema-fieldbuilder-one-shot
    relation: contrasts-with
---

# 共享参数不能同时绑 Instance 和 Type（Insert 静默返回 false）

## R — 原文 (Reading)

> 参数定义不能和实例和类型同时绑定。如果绑定参数已存在，则该方法返回 false。
>
> — 宦国胜, 第5章 5.1.3 (约p319)

---

## I — 方法论骨架 (Interpretation)

这是 Revit 共享参数绑定的一个"静默失败型"API 陷阱：

同一参数 Definition 不能既绑 Instance 又绑 Type——这是 Revit 参数模型的固有约束。但违反此约束时，`BindingMap.Insert` 不会抛异常，而是静默返回 false。开发者极易误以为绑定成功，导致参数在编辑器中的状态不可预期。

正确的防御做法：
1. 把 Insert 的 bool 返回值当作成功标志——`if (!bindingMap.Insert(def, binding)) { /* 处理失败 */ }`
2. 调用前用 `bindingMap.Contains(definition)` 预检是否已有绑定
3. 若已有绑定且需切换模式，先 Remove 再 Insert

这种"静默失败而非抛异常"的模式是 Revit API 中多类方法的共同特征，需要养成检查返回值的习惯。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 误插两次的防御

- **问题**: 同一参数定义被误插了两次（先 Type 后 Instance），代码不会报错但绑定状态不可预期
- **方法论的使用**: 识别这是"静默失败型"API——第二次 Insert 返回 false 不抛异常。防御：检查 bool 返回值 + 调用前 Contains 预检
- **结论**: 必须把 Insert 返回值当成功标志，不能假设 Insert 一定成功
- **结果**: 代码正确处理绑定失败，不会出现不可预期的参数状态

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 执行 BindingMap.Insert 后发现参数未出现在属性对话框中
2. 同一参数 Definition 被多次绑定（先 Type 后 Instance 或反之）
3. 编写健壮的共享参数绑定代码，需要防御性检查
4. 调试参数绑定失败的根因

### 语言信号 (用户的话里出现这些就应激活)

- "Insert 返回 false / insert returns false / 绑定失败"
- "静默失败 / silent failure / 不报错但没绑上"
- "同时绑 instance 和 type / 重复绑定 / duplicate binding"

### 与相邻 skill 的区分

- 与 `revit-schema-fieldbuilder-one-shot` 的区别：该 skill 是可扩展存储 builder 的一次性约束（违反会抛异常）；本 skill 是共享参数 Insert 静默返回 false（不抛异常），失败模式一明一暗。
- 与 `revit-binding-type-vs-instance` 的关系：本 skill 是该绑定选型后最常见陷阱的防御专题。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **调用前预检**
   - `if (bindingMap.Contains(definition))` 检查是否已有绑定
   - 若已有且需切换模式：先 `bindingMap.Remove(definition)` 再 Insert
   - 完成标准: 确认当前无冲突绑定

2. **执行 Insert 并检查返回值**
   - `bool success = bindingMap.Insert(definition, binding);`
   - `if (!success) { /* 绑定已存在或类型冲突，处理失败 */ }`
   - 完成标准: success 为 true
   - 判停条件: 若 success 为 false，说明该 Definition 已有绑定，停止并处理冲突

3. **验证绑定生效**
   - 取目标类别的实例/类型，确认参数出现在属性对话框中
   - 完成标准: 参数可见且值正确

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 可扩展存储——Schema/Entity 机制不走 BindingMap
- 首次绑定且确定无冲突——直接 Insert 即可，但仍建议检查返回值

### 作者在书中警告的失败模式

- Insert 静默返回 false 不抛异常——开发者误以为成功
- 类型/实例绑定二选一是固有约束，不能绕过

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能改善 API 反馈（如抛异常或提供更详细的失败信息）

### 容易混淆的邻近方法论

- 静默返回 false（Insert）vs 抛异常（SchemaBuilder.Finish 后改字段）——两种不同的失败反馈模式
- BindingMap.Contains 预检 vs 检查 Insert 返回值——前者是前置防御，后者是后置验证

---

## 相关 skills

- **revit-schema-fieldbuilder-one-shot**（反例：使用 SchemaBuilder.AddSimpleField 后再改字段 · contrasts-with）— 该 skill 是可扩展存储 builder 一次性约束（抛异常），本 skill 是共享参数 Insert 静默返回 false（不抛异常），失败模式一明一暗。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
