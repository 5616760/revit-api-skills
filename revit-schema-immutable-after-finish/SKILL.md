---
name: revit-schema-immutable-after-finish
description: |
  当可扩展存储的 Schema 已构建完成（Finish）后需要修改字段时调用。
  不适用于：首次构建 Schema、共享参数。
  关键 trigger 信号："Schema 不可改 schema immutable"、"Finish 后不能编辑 cannot edit after finish"、"加字段 add field"、"版本化 schema versioning"、"FieldBuilder 一次性"。
  核心认知：SchemaBuilder.Finish() 后 Schema 不可变，改字段只能建新 Schema 嵌版本号 + 数据迁移。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.4.1 (约p322)
tags: [schema, immutable, finish, versioning, extensible-storage]
related_skills: []
---

# SchemaBuilder.Finish() 之后 schema 不可再编辑

## R — 原文 (Reading)

> 使用 SchemaBuilder 的构造函数来创建新的架构。SchemaBuilder 类是一个用于创建架构的 helper 类。一旦 SchemaBuilder 完成了架构，就可以使用 Schema 类访问架构属性。到了这个阶段，架构就不可再编辑。
>
> — 宦国胜, 第5章 5.1.4.1 (约p322)

---

## I — 方法论骨架 (Interpretation)

可扩展存储的 Schema 是不可变对象。`SchemaBuilder.Finish()` 之后：

- Schema 不可再编辑——不能加字段、不能改字段类型、不能改权限
- FieldBuilder 是一次性 builder——完成后所有 `AddXxxField` / `SetUnitType` / `SetDocumentation` / `SetSubSchemaGUID` 不可用
- Schema 随文档保存，持久化后读取的也是不可变对象

这意味着 Schema 上线后要加字段，没有运行时 API 可改——只能：
1. 创建新 Schema，在名称里嵌入版本号（如 `WireSpliceLocation_v2`）
2. 从旧 Entity 读数据，写入新 Schema 的 Entity
3. 用 SchemaList 维护 v1→v2 兼容关系

版本化管理思路与共享参数 .txt 定义文件管理版本一致。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: Schema 上线后加字段

- **问题**: schema 上线后要加一个字段，且不丢老数据
- **方法论的使用**: 识别 Schema 不可变约束——没有运行时改字段 API。正确做法：创建新 Schema `Xxx_v2` 嵌版本号 + 数据迁移（读旧 Entity → 写新 Entity）+ SchemaList 维护兼容
- **结论**: 不能在原 Schema 上追加字段，必须建新 Schema
- **结果**: 老数据保留在新 Schema 中，版本兼容

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. Schema 上线后需要新增/修改字段
2. 运行时尝试调 AddSimpleField 报异常
3. 需要设计可扩展存储的版本化策略
4. 多版本 Schema 数据兼容管理

### 语言信号 (用户的话里出现这些就应激活)

- "Schema 不可改 / schema immutable / cannot edit after finish"
- "加字段 / add field to existing schema"
- "版本化 / versioning / schema version"
- "FieldBuilder 一次性 / fieldbuilder one-shot"

### 与相邻 skill 的区分

- 与 `revit-schema-fieldbuilder-one-shot` 的区别：该 skill 是 AddSimpleField 后再改字段的具体反例（抛异常行为）；本 skill 是 Schema 不可变的版本化管理策略。
- 与 `revit-extensible-storage-pipeline` 的区别：该 skill 是六步流水线全流程；本 skill 专注 Finish 后不可变约束。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认 Schema 确实已 Finish 且不可改**
   - 检查 Schema 是否已通过 `builder.Finish()` 完成
   - 完成标准: 确认 Schema 不可变，无法在原 Schema 上加字段

2. **创建新版本 Schema**
   - 在名称中嵌入版本号：`SchemaBuilder("MySchema_v2")`
   - 复制原字段 + 新增字段
   - 完成标准: 新 Schema 创建完成

3. **数据迁移与兼容维护**
   - 遍历挂载了旧 Schema Entity 的图元 → 读旧 Entity 数据 → 创建新 Entity 赋值 → SetEntity 挂载新 Entity
   - 用 SchemaList 维护 v1→v2 兼容关系
   - 完成标准: 所有数据迁移到新 Schema，旧 Entity 可清理

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 首次构建 Schema——在 Finish 前自由定义字段
- 共享参数——走 .txt 定义文件，不是 Schema

### 作者在书中警告的失败模式

- Finish 后再调 AddSimpleField → 抛异常
- FieldBuilder 完成后所有写操作不可用

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能提供 Schema 迁移工具或字段追加 API

### 容易混淆的邻近方法论

- SchemaBuilder（一次性 builder，完成后失效）vs 共享参数 .txt 定义文件（可随时编辑追加定义）——前者不可变，后者可变
- Schema 版本号命名 vs 共享参数 GUID——前者管版本，后者管唯一标识

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
