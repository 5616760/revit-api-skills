---
name: revit-assembly-part-modeling
description: |
  需要以编程方式做构造建模——创建部件（Assembly）、设置类型名、创建部件视图、把图元分割为零件（Part）时调用：按"创建→属性赋值→视图"三阶段分事务执行。Trigger：AssemblyInstance、CreateParts、AssemblyTypeName、部件视图、构造建模、part maker。不适用于：普通图元分组（Group 体系）、明细表统计本身。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.14 构造建模（约p407-409）
tags: [revit-api, assembly, part, construction-modeling, transaction]
related_skills:
  - slug: revit-partmaker-term
    relation: depends-on
---

# 构造建模（部件/零件/视图）决策流程

## R — 原文 (Reading)

> Revit API 允许将图元分割为子零件或将它们收集到部件中，以支持构造建模工作流，与 Revit 用户界面所能做的差不多。构造建模的主要类有：AssemblyInstance、AssemblyType、PartUtils、AssemblyViewUtils。
>
> — 宦国胜, 第5章 5.14 构造建模 约p407–p409

---

## I — 方法论骨架 (Interpretation)

构造建模 = 把图元**打散成零件**（Part）或**收集成部件**（Assembly），两条线各有专用类：`PartUtils`（零件）与 `AssemblyInstance` / `AssemblyType` / `AssemblyViewUtils`（部件）。零件/部件可独立列出明细、标记、过滤和输出，零件还能再分割。

关键执行纪律是**三阶段分事务**：

1. **事务 1——创建**：`AssemblyInstance.Create(doc, elementIds, categoryId)`（需有效 CategoryId 校验）创建部件实例；或 `PartUtils.CreateParts()` 创建零件（注意它实际创建的是 PartMaker，见零件术语 skill）。
2. **事务 2——属性赋值**：`AssemblyTypeName` 必须在部件实例已存在后才能设置。
3. **事务 3——视图**：用 `AllowsAssemblyViewCreation()` 先检查条件，再经 `AssemblyViewUtils` 创建部件视图；视图创建触发的文件重生成只能随独立事务提交进行。

跨章节印证：这与"创建层+创建轴网"合并撤销、几何/分析模型提交后才填充等时序约束同构——**依赖前序结果的写操作，各自独立事务**。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建部件后立即命名并建视图
- **问题**: 创建部件后想立即设置 AssemblyTypeName 并创建视图，为什么不能在一个事务中完成？
- **方法论的使用**: 按三阶段分事务——AssemblyTypeName 依赖部件实例已存在（事务 1 先建实例）；视图创建取决于 AllowsAssemblyViewCreation() 条件且其触发的重生成需独立事务提交。
- **结论**: "创建→属性赋值→视图"三阶段分事务是硬约束，不是风格偏好。
- **结果**: 分事务后命名与视图创建都成功，各阶段失败可独立回滚。

### 案例 2: 撤销菜单对齐 UI 体验
- **问题**: 多事务分阶段创建后，用户希望撤销时像 UI 操作一样一项搞定。
- **方法论的使用**: 借鉴"TransactionGroup 合并多事务为撤销菜单一项"的模式，把三个事务包进 TransactionGroup 并 Assimilate。
- **结论**: 阶段内事务独立保证时序，组级合并保证用户体验。
- **结果**: 撤销行为与 UI 手工构造建模一致。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写预制/深化工具：把相关图元收集为部件并自动生成部件视图（平面/立面/明细）。
2. 需要把墙/楼板分割为零件做分层材料统计（每层独立出量）。
3. 用户在一个事务里"创建+命名+建视图"失败，找原因。
4. 需要程序化判断某组图元能否创建部件视图。

### 语言信号 (用户的话里出现这些就应激活)

- "创建部件 / 部件视图" / "create assembly / AssemblyViewUtils"
- "分割为零件" / "split elements into parts / CreateParts"
- "AssemblyTypeName 设置失败 / 一个事务里做不完" / "set assembly type name fails"
- "构造建模" / "construction modeling / part maker"

### 与相邻 skill 的区分

- 与 `revit-partmaker-term` 的关系：该 skill 专讲零件“延迟生成”的术语机制；本 skill 是部件+零件+视图的全流程决策，依赖该术语理解。
- 与通用事务类 skill 的区别：事务类 skill 讲事务层级一般规则；本 skill 是该规则在构造建模上的具体三阶段应用。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **事务 1：创建**
   - 部件：`AssemblyInstance.Create(doc, elementIds, categoryId)`（先确认 categoryId 有效）；零件：`PartUtils.CreateParts(doc, elementIds)`。
   - 完成标准: 部件实例已存在 / PartMaker 已创建（零件本身等重生成后可见，见零件术语 skill）。
   - 判停条件: 若图元集合为空或 CategoryId 无效，先修正输入，Create 校验不通过不进入下一步。
2. **事务 2：属性赋值**
   - 新事务中设置 `AssemblyTypeName`（或后续需要的部件参数）。
   - 完成标准: 类型名设置成功且可读回。
3. **事务 3：视图（如需）**
   - 先 `AllowsAssemblyViewCreation()` 检查条件，再 `AssemblyViewUtils` 创建所需视图；提交事务触发重生成。
   - 完成标准: 视图创建成功（或条件不满足被明确报告）；如需单步撤销体验，将三事务包入 TransactionGroup 并 Assimilate。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 普通图元编组（Group 体系）——Group 不是部件，无部件视图能力。
- 只做明细表统计——零件/部件"可独立列明细"是特性，但明细表生成走明细表 API。
- 试图在一个事务里完成全部步骤——时序约束不允许。

### 作者在书中警告的失败模式

- 一个事务内"创建+命名+建视图"——AssemblyTypeName 设置与视图创建均可能失败（前序结果未落库/重生成无法进行）。
- 跳过 `AllowsAssemblyViewCreation()` 直接建视图——不满足条件的部件会失败。
- 忽视 CreateParts 的延迟生成语义——立即查询新零件得到空集合（见零件术语 skill）。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。构造建模是 2012 前后引入的功能，2014 API 覆盖有限；新版对零件分割、部件视图类型有扩充。
- `AssemblyInstance.Create` 需要有效 CategoryId 校验这类细节在新版签名有变化。

### 容易混淆的邻近方法论

- PartMaker 术语 skill——本流程第 1 步的底层机制。
- 事务分层方法论——本 skill 的通用上位规则。

---

## 相关 skills

- **revit-partmaker-term**（零件（Part）/ PartMaker · depends-on）— 本 skill 的零件创建依赖 PartMaker 的“延迟生成”术语机制。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
