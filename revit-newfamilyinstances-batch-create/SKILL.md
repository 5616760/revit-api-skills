---
name: revit-newfamilyinstances-batch-create
description: |
  用户要一次创建大量同类型族实例（如 1000 个柱子）、问批量创建怎么提速、或困惑 NewFamilyInstance 与 NewFamilyInstances 区别时调用。不适用于：实例数量少、参数差异大需逐个微调、放置 Tags 等特殊类别（有专用方法）。关键 trigger："一次创建 1000 个"、"batch create instances"、"NewFamilyInstances"、"FamilyInstanceCreationData"。核心：先加载符号 → 构造 N 个 FamilyInstanceCreationData 收进集合 → 一次 NewFamilyInstances 调用，减少事务内多次调用开销。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.3 族实例（p164–165）
tags: [newfamilyinstances, batch-create, creationdata, performance, bulk-placement, revit-api]
related_skills:
  - slug: revit-loadfamilysymbol-preference
    relation: composes-with
---

# 优先使用 NewFamilyInstances 批量创建族实例以提高性能

## R — 原文 (Reading)

> 一些 FamilyInstance 对象需要创建多个位置，在这种情况下，使用该对象所提供的更详细的创建方法更为适合……使用 Document.NewFamilyInstances()，通过一次创建多个族实例，可以精简代码并提高性能。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.3 族实例（p164–165）

---

## I — 方法论骨架 (Interpretation)

要放 N 个同类型族实例，别在循环里逐个调 NewFamilyInstance——用**批量通道**：

1. **先加载符号**：LoadFamilySymbol 把目标类型带进项目。
2. **收集数据**：构造 N 个 `FamilyInstanceCreationData` 对象（位置、符号、标高、结构类型等），收进一个集合——数据类把每个实例的参数"打包"。
3. **一次调用**：`NewFamilyInstances(creationDataCollection)` 一次创建全部。

收益：精简代码 + 减少事务内多次调用开销（事务里逐条创建是常见性能杀手）。

这是全书的一贯模式——"**复数形 API 一次处理集合**"：
- `document.Create.NewGrids(curveArray)` 批量轴网；
- `ElementTransformUtils.RotateElements / MirrorElements` 批量变换。

**例外规则**：某些特殊类别有专用创建方法，不能塞进批量族实例接口——如 Tags 必须走 `NewTag`。批量前先确认目标类别是否有专用通道。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量放柱/构件
- **问题**: 事务里要创建上千个同类型构件，逐条创建太慢。
- **方法论的使用**: 构造 FamilyInstanceCreationData 集合 → 一次 NewFamilyInstances。
- **结论**: 批量接口在大量同型实例场景显著提速。
- **结果**: 代码精简、性能提升。

### 案例 2: 批量轴网
- **问题**: 要一次性创建多条轴网。
- **方法论的使用**: NewGrids(curveArray) 批量创建。
- **结论**: "复数 API 批量处理集合"是全书的组织模式。
- **结果**: 一次调用完成多轴网。

### 案例 3: 批量变换
- **问题**: 对一批构件同时旋转/镜像。
- **方法论的使用**: RotateElements / MirrorElements 复数版本。
- **结论**: 批量变换与批量创建共享同一范式。
- **结果**: 一次调用处理整个集合。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写批量放置工具（柱网、座椅排布、灯光阵列）。
2. 事务中循环创建太慢，需要性能优化。
3. 问 NewFamilyInstance 和 NewFamilyInstances 该用哪个。
4. 需要给每个实例配不同位置/标高参数。

### 语言信号 (用户的话里出现这些就应激活)

- "一次创建 1000 个"（"create 1000 instances at once"）
- "批量创建族实例"（"batch create family instances"）
- "NewFamilyInstances / FamilyInstanceCreationData"
- "创建太慢怎么优化"（"placement too slow"）
- "批量放柱子/座位/灯具"

### 与相邻 skill 的区分

- 与 `revit-loadfamilysymbol-preference`：批量放置前需先加载符号，组合其加载策略——加载在前、放置在后。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认适用性**
   - 实例数量是否较多、类型是否相同、参数结构是否统一？
   - 特殊类别（如 Tags）？→ 查专用方法，不走批量接口。
   - 完成标准: 明确"可批量"且"无专用通道例外"。

2. **准备数据集合**
   - 先加载符号（LoadFamilySymbol）。
   - 构造 N 个 FamilyInstanceCreationData（位置/符号/标高/结构类型）收进集合。
   - 完成标准: 数据集合完整、字段与目标重载匹配。

3. **一次调用并验证**
   - `NewFamilyInstances(collection)` 在事务中调用，提交。
   - 抽查结果数量与位置正确。
   - 完成标准: 批量创建成功，实例数量与预期一致。
   - 判停条件: 若数量少或每个实例参数差异大需逐个微调，改用单数 NewFamilyInstance。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 实例数量少（个位数），批量化的收益可忽略。
- 每个实例的参数差异大、需要逐一逻辑控制。
- Tags 等有专用创建方法的类别——塞进批量接口会失败。

### 作者在书中警告的失败模式

- Tags 等特殊类别必须走专用方法（NewTag），不能塞进批量族实例接口。
- 逐条创建造成事务内多次调用——性能问题的根源之一。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：批量 API 变体（NewFamilyInstances2 等）在后续版本扩展，但"数据类收集 + 一次调用"的范式延续。

### 容易混淆的邻近方法论

- 批量创建（NewFamilyInstances）与批量变换（RotateElements/MirrorElements）是"创建"与"后处理"两个阶段。
- FamilyInstanceCreationData 只是"数据载体"，不是图元——别试图对它做图元操作。

---

## 相关 skills

- **revit-loadfamilysymbol-preference**（composes-with）：批量放置前需先加载符号，组合其加载策略——加载在前、放置在后。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
