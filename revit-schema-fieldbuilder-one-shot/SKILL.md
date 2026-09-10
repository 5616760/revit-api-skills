---
name: revit-schema-fieldbuilder-one-shot
description: |
  当 SchemaBuilder.Finish() 后尝试修改字段（AddSimpleField 等）并遇到异常时调用。
  不适用于：Finish 前字段定义、共享参数。
  关键 trigger 信号："AddSimpleField 后再改 fieldbuilder after finish"、"SchemaBuilder 抛异常"、"一次性 builder one-shot builder"。
  核心：FieldBuilder 一次性，Finish 后所有写操作抛异常，正确做法是建新 Schema 做迁移。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.4.2 (约p323)
tags: [schema, fieldbuilder, one-shot, counter-example, extensible-storage]
related_skills:
  - slug: revit-schema-immutable-after-finish
    relation: depends-on
---

# 反例：使用 SchemaBuilder.AddSimpleField 后再改字段

## R — 原文 (Reading)

> 一旦用 SchemaBuilder 完成了架构，字段就不再可用 FieldBuilder 进行编辑。
>
> — 宦国胜, 第5章 5.1.4.2 (约p323)

---

## I — 方法论骨架 (Interpretation)

这是可扩展存储的严格"一次性 builder"约束反例：

`SchemaBuilder.Finish()` 完成后，所有通过 `AddSimpleField` / `AddArrayField` / `AddMapField` 创建的字段不可再用 `FieldBuilder` 编辑。具体来说：

- `SetUnitType` / `SetDocumentation` / `SetSubSchemaGUID` 必须在 Finish 前设好
- Finish 后再调任何 `AddXxxField` 或 FieldBuilder 写方法 → 抛异常
- SchemaBuilder 完成后失效，不再是可操作的 builder 对象

这种"builder 一次性、完成后所有写操作抛异常"的设计在常规 ORM/序列化框架（如 JSON Schema、Entity Framework）中罕见——通常这些框架允许运行时修改字段定义。Revit 可扩展存储的不可变约束是特有的。

正确做法：创建新 Schema（嵌版本号命名如 `Xxx_v2`）+ 数据迁移，与共享参数 .txt 定义文件的版本管理思路一致。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 运行时追加字段抛异常

- **问题**: 运行时想给已有 schema 追加一个字段，直接再调 AddSimpleField 会发生什么？
- **方法论的使用**: 识别"一次性 builder"约束——Finish 后 SchemaBuilder 失效，所有 AddXxxField 不可用，抛异常。正确做法是创建 v2 schema 并做数据迁移
- **结论**: 不能在已有 Schema 上追加字段，必须建新 Schema
- **结果**: 通过版本化 Schema + 数据迁移解决字段追加需求

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 运行时尝试给已有 Schema 追加字段并遇到异常
2. 设计可扩展存储架构时需要理解 builder 的生命周期
3. 需要在 Finish 前设好所有字段属性（SetUnitType/SetDocumentation/SetSubSchemaGUID）
4. 调试 SchemaBuilder 相关异常

### 语言信号 (用户的话里出现这些就应激活)

- "AddSimpleField 后再改 / fieldbuilder after finish"
- "SchemaBuilder 抛异常 / schema builder exception"
- "字段不可编辑 / field not editable"
- "一次性 builder / one-shot builder"

### 与相邻 skill 的区分

- 与 `revit-schema-immutable-after-finish` 的关系：本 skill 是 AddSimpleField 后再改字段的具体反例（抛异常）；该 skill 是 Schema 不可变的版本化管理策略。
- 与 `revit-extensible-storage-pipeline` 的区别：该 skill 是六步流水线全流程；本 skill 是其中 Finish 后误操作的防御。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认异常来源是一次性 builder 约束**
   - 检查是否在 Finish 后调用了 AddXxxField 或 FieldBuilder 写方法
   - 确认 SetUnitType/SetDocumentation/SetSubSchemaGUID 是否在 Finish 前设好
   - 完成标准: 确认异常由 Finish 后写操作触发

2. **改为创建新版本 Schema**
   - 创建 `SchemaBuilder("Xxx_v2")`，包含原字段 + 新字段
   - 在 Finish 前设好所有字段属性
   - 完成标准: 新 Schema 创建完成

3. **数据迁移**
   - 遍历挂载旧 Entity 的图元 → 读旧数据 → 创建新 Entity 赋值 → SetEntity
   - 完成标准: 数据迁移完成，旧 Entity 可清理

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- Finish 前的字段定义——此时 AddXxxField 完全可用
- 共享参数——走 .txt 定义文件，可随时编辑

### 作者在书中警告的失败模式

- Finish 后再调 AddSimpleField → 抛异常
- Finish 后 FieldBuilder 所有写方法不可用
- SetUnitType/SetDocumentation/SetSubSchemaGUID 未在 Finish 前设好 → 丢失或报错

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能提供 Schema 字段追加或迁移工具

### 容易混淆的邻近方法论

- SchemaBuilder 一次性（抛异常）vs BindingMap.Insert 静默返回 false（不抛异常）——两种不同的失败反馈模式
- FieldBuilder 一次性 vs SchemaBuilder 一次性——前者管字段，后者管架构

---

## 相关 skills

- **revit-schema-immutable-after-finish**（SchemaBuilder.Finish() 之后 schema 不可再编辑 · depends-on）— 本 skill 是 Finish 后再改字段的反例，依赖该 skill 的 schema 不可变原则。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
