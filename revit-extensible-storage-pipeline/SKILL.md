---
name: revit-extensible-storage-pipeline
description: |
  当需要在 Revit 图元上存储结构化、版本化、权限控制的隐藏数据时调用六步流水线。
  不适用于：需要出现在属性窗口/明细表/ODBC 导出的数据（用共享参数）、临时内存数据。
  关键 trigger 信号："可扩展存储 extensible storage"、"Schema Entity"、"隐藏数据结构 hidden structured data"、"读写权限 access level"、"第三方程序数据 third-party data"。
  六步：建 Schema → 设权限 → 定义字段 → 创建 Entity → 赋值 → 挂载图元。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.4 (约p322-324)
tags: [extensible-storage, schema, entity, access-level, revit-api]
related_skills:
  - slug: revit-schema-immutable-after-finish
    relation: composes-with
---

# 可扩展存储（Extensible Storage）六步流水线

## R — 原文 (Reading)

> 随 Revit 图元存储数据需按以下步骤完成：(1) 创建并命名新架构。(2) 设置架构的读/写访问权限。(3) 定义架构的一个或多个数据字段。(4) 根据架构创建条目。(5) 对条目的字段赋值。(6) 将条目关联到 Revit 图元。
>
> — 宦国胜, 第5章 5.1.4 (约p322-324)

---

## I — 方法论骨架 (Interpretation)

可扩展存储是 Revit 在图元上存储结构化隐藏数据的机制，按固定六步流水线执行：

1. **建 Schema**：用 `SchemaBuilder(schemaName)` 创建新架构并命名。
2. **设权限**：`SetReadAccessLevel` / `SetWriteAccessLevel` 设读写权限（Public / Vendor / Application），写访问非 Public 时必须 `SetVendorId`。
3. **定义字段**：用 `AddSimpleField<T>` / `AddArrayField<T>` / `AddMapField<K,V>` 定义字段；带单位的字段必须给 `DisplayUnit` 参数。
4. **创建 Entity**：`schema = builder.Finish()` 完成 Schema 后，`new Entity(schema)` 创建条目。
5. **赋值**：`entity.Set<T>(fieldName, value)` 或 `entity.Set<T>(field, value, DisplayUnitType)` 给字段赋值。
6. **挂载图元**：`element.SetEntity(entity)` 将条目关联到图元。

所有修改必须在 Transaction 内。Schema 一旦 Finish 不可再改。版本化只能在名称里嵌版本号，创建新 Schema 做数据迁移。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 端到端创建→赋值→读取

- **问题**: 需要在管道图元上存储结构化检查记录（日期+检查人+历史数组）
- **方法论的使用**: 按六步流水线——建 Schema "PipeInspection" → 设 Public 读 / Vendor 写 → 定义 string 日期 + string 检查人 + Array<string> 历史 → 创建 Entity → 赋值 → SetEntity 挂载
- **结论**: 数据结构化存储且对用户不可见，第三方程序可按权限读写
- **结果**: 管道检查记录成功存储，属性窗口中不可见

### 案例 2: 读写权限配置

- **问题**: 同一 schema 希望第三方厂商程序也能读但只有自己能写
- **方法论的使用**: `SetReadAccessLevel(AccessLevel.Public)` + `SetWriteAccessLevel(AccessLevel.Vendor)` + `SetVendorId("myVendorId")`
- **结论**: 第三方可读不可写，写入权限锁定到 Vendor
- **结果**: 权限隔离成功，其他厂商程序可读取但不能篡改

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要在图元上存储结构化/嵌套数据（数组、Map），共享参数无法满足
2. 需要数据对 Revit UI 完全隐藏（不在属性窗口/明细表出现）
3. 需要读写权限控制——第三方程序可读不可写
4. 需要版本化数据结构管理

### 语言信号 (用户的话里出现这些就应激活)

- "可扩展存储 / extensible storage / Schema Entity"
- "隐藏数据结构 / hidden structured data"
- "读写权限 / access level / read write permission"
- "第三方程序数据 / third-party data storage"
- "SchemaBuilder / 建架构"

### 与相邻 skill 的区分

- 与 `revit-schema-immutable-after-finish` 的关系：本 skill 是六步流水线全流程；该 skill 专注 Finish 后不可变约束与版本化应对，是本流程的关键约束。
- 与 `revit-data-storage-paths` 的区别：该 skill 是共享参数 vs 可扩展存储的选型决策；本 skill 是选定可扩展存储后的六步实现。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **建 Schema 并设置权限与字段**
   - `SchemaBuilder builder = new SchemaBuilder(schemaGuid);`
   - `builder.SetReadAccessLevel(...)` / `builder.SetWriteAccessLevel(...)` / `builder.SetVendorId(...)`（写访问非 Public 时必须设）
   - `builder.AddSimpleField<T>("fieldName")` 或 `AddArrayField<T>` / `AddMapField`；带单位的字段必须给 DisplayUnit 参数
   - `Schema schema = builder.Finish();`
   - 完成标准: Schema 创建完成且不可再编辑
   - 判停条件: 若 Finish 前漏设 VendorId（写访问非 Public），会报错，补设后重试

2. **创建 Entity 并赋值**
   - `Entity entity = new Entity(schema);`
   - `entity.Set<T>("fieldName", value);` 带单位的用 `entity.Set<T>(field, value, DisplayUnitType)`
   - 完成标准: Entity 所有字段已赋值

3. **在 Transaction 内挂载到图元**
   - `using (Transaction t = new Transaction(doc, "store data")) { t.Start(); element.SetEntity(entity); t.Commit(); }`
   - 完成标准: Entity 成功关联到图元，可通过 `element.GetEntity(schema)` 读取验证

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要数据出现在明细表/tag/属性窗口/ODBC 导出——用共享参数，可扩展存储对 UI 完全不可见
- 简单键值数据且无需隐藏——共享参数更轻量

### 作者在书中警告的失败模式

- 写访问非 Public 时未设 VendorId → Finish 前报错
- 带单位的字段未给 DisplayUnit 参数 → 抛"单位缺失"异常
- 所有修改不在 Transaction 内 → 回滚丢弃 Schema 与 Entity
- Schema 一旦 Finish 不可改字段 → 要加字段只能建新 Schema（见 revit-schema-immutable-after-finish）

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，可扩展存储是较新 API，后续版本可能放宽字段类型限制或新增迁移工具

### 容易混淆的邻近方法论

- Schema（数据架构定义）vs Entity（数据实例）——前者是模板，后者是实例
- AccessLevel.Public（所有人可读写）vs AccessLevel.Vendor（仅 VendorId 匹配者可写）——权限粒度不同

---

## 相关 skills

- **revit-schema-immutable-after-finish**（SchemaBuilder.Finish() 之后 schema 不可再编辑 · composes-with）— 六步流程中 Finish 后 schema 不可再编辑，该 skill 提供版本化应对。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
