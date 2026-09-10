---
name: revit-partmaker-term
description: |
  术语 skill：PartUtils.CreateParts() 并不直接创建零件，而是实例化 PartMaker（工厂图元），由它在重生成期间驱动产生零件——调用后必须等事务提交/Regenerate 才能查到 Part。Trigger：CreateParts 后查不到零件、PartMaker、延迟创建、工厂图元、GetAssociatedParts 返回空。不适用于：部件（Assembly）创建、视图创建。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.14.2 零件（约p409）
tags: [revit-api, part, partmaker, regenerate, deferred-creation]
related_skills: []
---

# 零件（Part）/ PartMaker

## R — 原文 (Reading)

> 在 Revit API 中，可使用 PartUtils 类将图元分割为零件。需要注意的是，与 API 中的大多数图元的创建方法不同，CreateParts() 实际上并不创建或恢复零件，更确切地说是实例化一个名为 PartMaker 的图元。
>
> — 宦国胜, 第5章 5.14.2 零件 约p409

---

## I — 方法论骨架 (Interpretation)

一个容易踩坑的**术语/机制**：`PartUtils.CreateParts()` 的方法名有误导性。

1. **名不副实**：与大多数图元创建方法不同，CreateParts 不直接创建零件——它实例化一个 **PartMaker**（"零件制造器"）图元。
2. **延迟生成**：真正的 Part 由 PartMaker 按其内嵌规则在**重生成期间**驱动产生。调用 CreateParts 后立即查询零件，得到 null 或空集合。
3. **正确时序**：CreateParts → 事务提交（触发自动 Regenerate）或手动 Regenerate → 之后才能通过 `GetAssociatedParts` 查到实际的 Part 图元。
4. **模式识别**：这是"工厂图元"模式——一个图元充当生成器，产出另一批图元；类似时序约束的还有"几何/分析模型在事务提交后才填充"。
5. **成本意识**：重生成有代价，批量创建零件时避免频繁手动 Regenerate；也要注意 Regenerate 失败可能留下半完成状态。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: CreateParts 后立即查询零件
- **问题**: 调用 PartUtils.CreateParts 后立即查询新创建的零件，会得到什么结果？
- **方法论的使用**: 按延迟生成机制预判——得到 null 或空集合，因为 CreateParts 只创建了 PartMaker，零件要等下次重生成才产生。
- **结论**: 必须在事务提交触发自动 Regenerate（或手动 Regenerate）之后，才能用 GetAssociatedParts 查到 Part。
- **结果**: 调整时序后零件可正常检索，统计/出量流程跑通。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写分层出量工具（墙/板分割为零件后逐层统计材料），创建后查不到零件。
2. 调试"CreateParts 之后集合是空的"这类时序问题。
3. 用户读到 PartMaker 类型名，想知道它和 Part 的关系。
4. 设计零件再分割（零件分割为更小零件）的流程时需要理解生成机制。

### 语言信号 (用户的话里出现这些就应激活)

- "CreateParts 之后查不到零件" / "no parts after CreateParts / GetAssociatedParts empty"
- "PartMaker" / "零件制造器"
- "零件什么时候生成 / 延迟创建" / "when are parts created / deferred"
- "分割图元为零件" / "split element into parts"

### 与相邻 skill 的区分

- 与 `revit-assembly-part-modeling` 的区别：该 skill 是构造建模全流程（部件+零件+视图）；本 skill 只讲零件创建的“延迟生成”机制这一个术语点。
- 与其他“操作后需 Regenerate 才能访问结果”类 skill 的区别：同为时序约束，但本 skill 的机制是 PartMaker 驱动，不是属性回填。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **创建**
   - `PartUtils.CreateParts(doc, elementIds)`（一个或多个源图元）。
   - 完成标准: 调用成功返回，明确此刻只有 PartMaker 存在。
2. **等待重生成**
   - 提交事务（触发自动 Regenerate）；批量场景避免在事务内频繁手动 Regenerate。
   - 完成标准: 文档已重生成（提交完成）。
   - 判停条件: 若 Regenerate 失败，检查模型有效性——可能留下半完成状态，需评估回滚。
3. **检索与消费**
   - `GetAssociatedParts(...)` 查询实际 Part 图元；需要时继续分割（零件可再分割）。
   - 完成标准: 零件集合非空且与源图元关联正确，可交给统计/标记/输出流程。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 部件（Assembly）创建——AssemblyInstance.Create 是立即创建，无此延迟机制。
- 普通图元的"参数提交后才可见"问题——那是通用时序约束，不是 PartMaker 机制。

### 作者在书中警告的失败模式

- CreateParts 后立即查询零件——null/空集合，误以为创建失败而重复创建。
- 在事务内反复手动 Regenerate——重生成成本高，应依赖提交时的自动重生成。
- 忽视 Regenerate 失败风险——可能留下半完成状态且未回滚。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中零件 API（如零件分割参数化、形状驱动的零件）有扩展，PartMaker 机制本身延续。
- 书中未深入 PartMaker 内嵌规则的可配置性，复杂分割需求需另查文档。

### 容易混淆的邻近方法论

- 构造建模全流程 skill——本术语是其第 1 步机制。
- "事务提交后才能访问几何/分析模型"——相似时序、不同对象。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
