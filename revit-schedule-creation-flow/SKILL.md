---
name: revit-schedule-creation-flow
description: |
  当需要通过 API 创建明细表/材料提取/图纸列表等视图并配置字段/样式/分组/过滤时调用。
  不适用于：普通视图创建、视图分类判别。
  关键 trigger 信号："创建明细表 create schedule"、"材料提取 material takeoff"、"公式列 formula field"、"字段配置 schedule fields"、"表格数据 table data"、"行起始编号 first row number"。
  核心流程：选工厂方法建表 → GetSchedulableFields/AddField 配字段 → 样式/分组/过滤 → GetTableData 读数据。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.6.2节明细表视图 (约p134-140)
tags: [schedule, view-schedule, schedulable-fields, table-data, revit-api]
related_skills:
  - slug: revit-2d-view-creation-paths
    relation: contrasts-with
---

# 明细表创建与字段配置流程

## R — 原文 (Reading)

> ViewSchedule.CreateSchedule() 可以创建标准的单一类别或多类别明细表……ScheduleDefinition 类包含各种设置，定义明细表视图中的内容……使用 ScheduleDefinition.AddField() 方法，将字段添加到字段列表的末尾。
>
> — 宦国胜, 第2章 2.6.2节明细表视图 (约p134-140)

---

## I — 方法论骨架 (Interpretation)

明细表即视图（ViewSchedule 派生自 View），创建与配置按完整决策链执行：

1. **选工厂方法建表**：9 种明细表各有专用工厂方法——
   - `CreateSchedule`（标准明细表）/ `CreateAreaSchedule`（面积）/ `CreateKeySchedule`（关键字）/ `CreateMaterialTakeoff`（材料提取）/ `CreateViewList`（视图列表）/ `CreateSheetList`（图纸列表）/ `CreateKeynoteLegend`（关键字图例）/ `CreateNoteBlock`（注释块）/ `CreateRevisionSchedule`（修订明细表）
2. **配字段**：`ScheduleDefinition.GetSchedulableFields()` 获取可用字段 → `AddField(fieldId)` 添加；公式列通过 `AddField` 添加 `ScheduleFieldType.Formula` 类型字段。
3. **样式**：`TableCellStyle` + `SetFormatOptions` 配置单元格样式。
4. **分组排序**：`ScheduleSortGroupField` 配置分组排序规则。
5. **过滤**：`ScheduleFilter` 配置过滤条件。
6. **表格数据**：`GetTableData()` → `GetSectionData(sectionType)` 读取表格区域数据。

关键陷阱：行/列起始编号可能为 0 或 1，必须先读 `FirstRowNumber` / `FirstColumnNumber` 再定位单元格——直接硬编码索引会在不同区域设置下错位。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 带公式列的材料提取明细表

- **问题**: 如何创建带公式列的材料提取明细表？
- **方法论的使用**: 材料提取用 `CreateMaterialTakeoff`（不是 CreateSchedule 加参数）；公式列通过 `AddField` 添加 `ScheduleFieldType.Formula` 字段
- **结论**: 材料提取有专用工厂方法，不能误用 CreateSchedule
- **结果**: 成功创建带公式列的材料提取明细表

### 案例 2: 表格行号错位

- **问题**: 表格代码在某些版本读第 0 行、某些版本读第 1 行错位
- **方法论的使用**: 行列起始编号可能为 0 或 1，必须先读 `FirstRowNumber` / `FirstColumnNumber` 再定位单元格
- **结论**: 直接硬编码索引会在不同区域设置下错位
- **结果**: 改为先读起始编号再定位，错位问题解决

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要通过 API 自动创建明细表/材料提取/图纸列表
2. 需要配置明细表字段、样式、分组排序、过滤
3. 表格数据读取出现行号错位
4. 需要添加公式列到明细表
5. 需要区分 9 种明细表类型选择正确工厂方法

### 语言信号 (用户的话里出现这些就应激活)

- "创建明细表 / create schedule / ViewSchedule"
- "材料提取 / material takeoff / CreateMaterialTakeoff"
- "公式列 / formula field / ScheduleFieldType"
- "字段配置 / schedule fields / AddField / GetSchedulableFields"
- "表格数据 / table data / FirstRowNumber"

### 与相邻 skill 的区分

- 与 `revit-2d-view-creation-paths`：二者同属视图创建但互为分支——本 skill 管明细表类视图（CreateSchedule/CreateMaterialTakeoff），后者管平面/剖面/立面/图纸，按视图族选择。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **选工厂方法创建明细表**
   - 标准明细表：`ViewSchedule.CreateSchedule(doc, categoryId)`
   - 材料提取：`ViewSchedule.CreateMaterialTakeoff(doc, categoryId)`
   - 关键字：`ViewSchedule.CreateKeySchedule(doc, categoryId)`
   - 图纸列表：`ViewSchedule.CreateSheetList(doc)`
   - 视图列表：`ViewSchedule.CreateViewList(doc)`
   - 完成标准: ViewSchedule 对象创建成功

2. **配置字段、样式、分组、过滤**
   - `ScheduleDefinition def = schedule.Definition;`
   - `def.GetSchedulableFields()` → `def.AddField(fieldId)`
   - 公式列：`def.AddField(ScheduleFieldType.Formula, ...)`
   - `ScheduleSortGroupField` / `ScheduleFilter` 配置分组过滤
   - 完成标准: 字段和配置完成

3. **读取表格数据时先查起始编号**
   - `TableData data = schedule.GetTableData();`
   - `int firstRow = data.GetSectionData(SectionType.Body).FirstRowNumber;`
   - `int firstCol = data.GetSectionData(SectionType.Body).FirstColumnNumber;`
   - 用 firstRow/firstCol 偏移定位单元格
   - 完成标准: 单元格数据正确读取
   - 判停条件: 若硬编码索引 0 或 1 导致错位，改为先读 FirstRowNumber/FirstColumnNumber

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 普通视图创建（平面/剖面/立面）——走对应 Create 方法
- 视图分类判别——走 revit-view-classification-paths

### 作者在书中警告的失败模式

- 材料提取误用 CreateSchedule 而非 CreateMaterialTakeoff → 创建结果不正确
- 表格行/列起始编号不定（0 或 1）→ 硬编码索引在不同区域设置下错位

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增明细表类型或字段 API

### 容易混淆的邻近方法论

- CreateSchedule（标准明细表）vs CreateMaterialTakeoff（材料提取）——前者按类别，后者按材料
- ScheduleDefinition（明细表定义）vs TableData（表格数据）——前者管字段配置，后者管数据读取

---

## 相关 skills

- **revit-2d-view-creation-paths**（contrasts-with）：明细表创建与普通二维视图创建是视图族层面的对立分支，按目标视图类型择一。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
