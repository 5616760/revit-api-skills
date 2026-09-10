---
name: revit-shared-parameter-guid-check
description: |
  当复制带共享参数的图元后需要区分原始与副本、或需要检查共享参数定义来源时调用。
  不适用于：可扩展存储、首次创建共享参数。
  关键 trigger 信号："复制图元参数 copy element with parameter"、"GUID 一致 GUID same after copy"、"区分原始和副本 distinguish original and copy"、"ElementId UniqueId"、"参数定义级唯一 definition-level unique"。
  核心认知：共享参数 GUID 是定义级唯一，不随实例复制变化；区分实例需附加 Element.UniqueId。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录B FAQ B.1 (约p426)
tags: [shared-parameter, guid, copy, unique-id, data-lineage]
related_skills: []
---

# 共享参数复制需检查 GUID

## R — 原文 (Reading)

> 问：共享参数值会随相应的图元一起拷贝吗？答：是的。如果数据库中有一个共享参数，它拥有唯一图元 ID，请附加 Revit 图元唯一 ID 或向 Revit 图元唯一 ID 添加另一个共享参数。这样做，可以检查并确保正在使用的是原始图元 ID，而不是其拷贝。
>
> — 宦国胜, 附录B FAQ B.1 (约p426)

---

## I — 方法论骨架 (Interpretation)

共享参数的 GUID 有两个层次的唯一性，理解层次差异是数据血缘追踪的关键：

**定义级唯一（GUID）**：共享参数的 GUID 标识"参数定义"本身，不随实例复制而改变。复制一面带共享参数的墙后，副本的共享参数 GUID 与原始相同——因为它们引用的是同一个参数定义。

**实例级唯一（UniqueId）**：Element.UniqueId 标识"参数实例所在的图元"，每个图元有独立的 UniqueId。原始墙和副本墙的 UniqueId 不同。

因此：
- GUID 标识"这是什么参数"（定义级）
- UniqueId 标识"这个参数值在哪个图元上"（实例级）
- 两者组合可确定参数的唯一实例

添加新共享参数时应检查 GUID 确保使用的是原始定义而非意外拷贝。这是 Revit BIM 数据血缘追踪（data lineage）的特有机制。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 区分原始墙和副本墙

- **问题**: 复制一面带共享参数的墙后，副本的共享参数 GUID 与原始相同吗？如何区分原始和副本？
- **方法论的使用**: 识别 GUID 的定义级唯一性——GUID 不随实例复制变化。区分原始和副本需附加 Element.UniqueId：GUID 标识"参数定义"，UniqueId 标识"参数实例所在的图元"
- **结论**: GUID 相同（同一定义），UniqueId 不同（不同图元）。两者组合确定参数唯一实例
- **结果**: 成功区分原始墙和副本墙的参数实例

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 复制带共享参数的图元后需要区分原始和副本
2. 数据血缘追踪——确认使用的是原始定义而非意外拷贝
3. 批量复制图元后需要审计参数来源
4. 添加新共享参数时需要检查 GUID 唯一性

### 语言信号 (用户的话里出现这些就应激活)

- "复制图元参数 / copy element with parameter"
- "GUID 一致 / GUID same after copy / 副本 GUID 相同"
- "区分原始和副本 / distinguish original and copy"
- "ElementId UniqueId / 图元唯一 ID"
- "参数定义级唯一 / definition-level unique"

### 与相邻 skill 的区分

- 与 `revit-shared-parameter-file-swap` 的区别：该 skill 关注参数文件隔离管理；本 skill 关注复制后 GUID 的定义级唯一性与实例区分。
- 与 `revit-parameter-index-lookup` 的区别：该 skill 关注检索入口选择；本 skill 关注 GUID 在复制场景下的语义层次。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **获取共享参数的 GUID 和图元的 UniqueId**
   - `Guid paramGuid = parameter.GUID;`（定义级标识）
   - `string elementId = element.UniqueId;`（实例级标识）
   - 完成标准: 获取到 GUID 和 UniqueId

2. **比较 GUID 确认参数定义来源**
   - 原始与副本的 GUID 相同 → 同一参数定义
   - 若 GUID 不匹配 → 可能是意外拷贝的错误定义
   - 完成标准: 确认参数定义来源

3. **用 UniqueId 区分图元实例**
   - 原始与副本的 UniqueId 不同 → 不同图元实例
   - 两者组合（GUID + UniqueId）确定参数的唯一实例
   - 完成标准: 成功区分原始和副本

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 可扩展存储——走 Schema GUID，机制不同
- 首次创建共享参数——无需检查复制来源

### 作者在书中警告的失败模式

- 误以为副本的 GUID 与原始不同 → 导致错误的数据血缘判断
- 仅用 GUID 区分实例 → 无法区分原始和副本（GUID 相同）

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能提供更完善的参数血缘追踪 API

### 容易混淆的邻近方法论

- 共享参数 GUID（定义级唯一）vs Element.UniqueId（实例级唯一）——前者标识参数定义，后者标识图元实例
- 共享参数 GUID vs 可扩展存储 Schema GUID——两者都有 GUID 但用途不同

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
