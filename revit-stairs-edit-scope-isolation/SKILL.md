---
name: revit-stairs-edit-scope-isolation
description: |
  反例 skill：在 StairsEditScope 楼梯编辑会话内调用 Railing.Create 会失败——栏杆创建被隔离在会话之外，必须在普通 Transaction 中执行。Trigger：楼梯编辑会话里创建栏杆失败、StairsEditScope、Railing.Create、栏杆自动生成、"编辑上下文隔离"。不适用于：只读访问楼梯构件、普通图元的事务管理问题。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.10.1（约p256）
tags: [revit-api, counter-example, stairs-edit-scope, railing, transaction]
related_skills:
  - slug: revit-stairs-object-model
    relation: depends-on
---

# 反例：栏杆与楼梯编辑作用域的隔离限制

## R — 原文 (Reading)

> 在使用 StairsEditScope.Start（ElementId，ElementId）方法创建新楼梯时，会带有与其关联的默认栏杆扶手。不同于梯段和平台创建需要使用 StairsEditScope，在打开的楼梯编辑会话内是无法执行栏杆创建的。
>
> — 宦国胜, 第3章 3.10.1 楼梯（约p256）

---

## I — 方法论骨架 (Interpretation)

这是"编辑上下文隔离"的典型反例，揭示 Revit 的设计规则：

1. **绑定关系**：楼梯构件（梯段/平台）的创建**只能**在 `StairsEditScope` 编辑会话内进行；会话内的事务对用户不可见，最终合并为一次撤销。
2. **隔离规则**：栏杆创建被**明确排除**在楼梯编辑会话之外——`Railing.Create()` 必须在普通 `Transaction` 中调用。在打开的 StairsEditScope 里创建栏杆会直接失败。
3. **正确流程**（"新建楼梯+自定义栏杆"需求）：
   - StairsEditScope 内建楼梯 → Commit 结束会话；
   - 普通事务里 `GetAssociatedRailings()` 找到默认栏杆并删除；
   - 同一普通事务里 `Railing.Create(stairs.Document, stairs.Id, railingTypeId, RailingPlacementPosition.Treads)` 新建。
4. **一般化规律**：不同图元族绑定不同编辑会话（类似 FamilyEditScope 等），把多种图元创建塞进同一个作用域是常见错误来源。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: "新建楼梯同时自动生成自定义栏杆"的失败与修复
- **问题**: 客户要求新建楼梯同时自动生成自定义栏杆，开发者在同一个 StairsEditScope 里顺带 `Railing.Create()`。
- **方法论的使用**: 识别为编辑作用域隔离——梯段/平台在会话内建，会话 Commit 后开普通事务，先 `GetAssociatedRailings()` 删默认栏杆，再 `Railing.Create(...)` 新建。
- **结论**: Revit 按"图元族绑定编辑会话"划分世界，混用即失败。
- **结果**: 按三步流程执行，楼梯与自定义栏杆都成功创建，撤销菜单行为也正确（会话一项 + 栏杆事务）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在楼梯编辑会话内创建/修改栏杆，代码失败或抛异常。
2. 要实现"自动建楼梯并换成指定栏杆类型"的批量脚本，需要排事务顺序。
3. 需要理解为什么有的图元创建要 StairsEditScope、有的要普通事务。
4. 调试"撤销时楼梯和栏杆分成多项/顺序怪异"的行为。

### 语言信号 (用户的话里出现这些就应激活)

- "楼梯编辑会话里建栏杆失败" / "Railing.Create inside StairsEditScope fails"
- "StairsEditScope / 楼梯编辑作用域"
- "自动给楼梯配栏杆" / "auto-generate railing for stairs"
- "为什么梯段必须用 StairsEditScope" / "stairs edit scope transaction"

### 与相邻 skill 的区分

- 与 `revit-stairs-object-model` 的关系：该 skill 提供已存在楼梯的只读对象模型与构件导航；本 skill 是创建侧的作用域隔离反例（编辑会话内禁建栏杆），依赖其对象模型知识。
- 与通用事务类 skill 的区别：事务类 skill 讲 Transaction 的通用层级与模式；本 skill 讲特定图元族的编辑会话隔离，是更细的规则。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位失败原因**
   - 检查代码是否在 `StairsEditScope` 打开期间调用了 `Railing.Create()`（或其他会话外才允许的操作）。
   - 完成标准: 已确认/排除"会话内建栏杆"这一错误模式。
2. **重排执行顺序**
   - StairsEditScope：只做梯段/平台/支撑创建 → Commit 结束会话。
   - 完成标准: 会话已提交，楼梯图元存在。
3. **普通事务处理栏杆**
   - 开普通 Transaction：`GetAssociatedRailings()` 取默认栏杆（如有）删除 → `Railing.Create(doc, stairs.Id, railingTypeId, RailingPlacementPosition.Treads)` 新建 → 提交。
   - 完成标准: 新栏杆创建成功且楼梯原有构件未被破坏；撤销行为符合预期。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只读访问楼梯构件（梯段/平台/支撑/栏杆）——不涉及编辑会话，走楼梯对象模型 skill。
- 普通（非楼梯）图元的事务问题——用通用事务方法论。
- 楼梯类型/几何修改——需区分是否要求进入编辑会话，但栏杆隔离规则只针对创建。

### 作者在书中警告的失败模式

- 在打开的 StairsEditScope 内调用 Railing.Create——被明确禁止，直接失败。
- 误以为 Start 新建楼梯时自带的默认栏杆可替换——需要普通事务里删旧建新，不能原地改类型。
- 混淆会话内事务（用户不可见、合并撤销）与普通事务（独立撤销项）的语义。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。按构件楼梯与 StairsEditScope 均为 2013 引入的新机制，2014 文档描述有限；新版对多楼层楼梯、栏杆创建 API 有扩展。
- 书中未系统对比其他 EditScope（FamilyEditScope 等）的隔离规则，泛化时需自行验证。

### 容易混淆的邻近方法论

- 事务层级（Transaction/TransactionGroup/Regenerate）——通用的"包裹"问题；本 skill 是"哪个作用域管哪类图元"的绑定问题。
- 楼梯构件访问模型——本 skill 的上游知识。

---

## 相关 skills

- **revit-stairs-object-model**（楼梯构件对象模型与访问路径（Stairs→Runs/Landings/Supports） · depends-on）— 本 skill 讲编辑作用域内的隔离限制，需先理解楼梯对象模型 Stairs→Runs/Landings/Supports。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
