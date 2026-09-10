---
name: revit-dwg-export-tables
description: |
  让 Revit→CAD（DWG/DXF/DGN）导出满足公司图层/线型/填充/字体/线宽标准：用 BaseExportOptions 五张映射表（ExportLayerTable 等）编程配置，用 GetPredefinedOptions 复用 UI 预设。信号："图层映射 / export layer table / category to layer mapping"、"导出标准 / ExportLayerInfo"。不适用：选择导出格式、IFC/gbXML 非 CAD 格式、自定义导出器。
  Trigger: GetPredefinedOptions / 图层映射 / layer mapping。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.16.1 导出表（约p421）
tags: [revit-api, dwg-export, export-tables, layer-mapping, cad-standards]
related_skills:
  - slug: revit-document-export-formats
    relation: depends-on
  - slug: revit-printable-view-check
    relation: composes-with
  - slug: revit-ifc-exporter-registration
    relation: contrasts-with
---

# DWG/DXF/DGN 导出选项表映射

## R — 原文 (Reading)

> 这些格式的每个导出选项派生自 BaseExportOptions，因此有许多共同的导出设置，例如颜色模式或是否隐藏范围框。BaseExportOptions 还有一个可以返回文件的任何预定义设置的静态方法 GetPredefinedSetupNames()。
>
> — 宦国胜, 第5章 5.16.1 导出表（约p421）

---

## I — 方法论骨架 (Interpretation)

Revit 导出到 CAD 格式时的"翻译规则"不是散装开关，而是**五张映射表**，每张表规定一类 Revit 概念到 CAD 概念的对应：

| 表 | 映射内容 |
|---|---|
| ExportLayerTable | Revit 类别/子类别 → CAD 图层（名称、颜色、剖切颜色） |
| ExportLinetypeTable | 线型对应 |
| ExportPatternTable | 填充图案对应 |
| ExportFontTable | 字体对应 |
| ExportLineweightTable | 线宽对应 |

操作套路分两层：

1. **拿选项**：三种 CAD 格式的导出选项都派生自 `BaseExportOptions`（共享颜色模式、隐藏范围框等通用设置）。优先用 `BaseExportOptions.GetPredefinedSetupNames(doc)` 列出文档已有预设名，再把名字传给对应选项类的静态 `GetPredefinedOptions(doc, setupName)` 取现成选项——相当于复用 UI 里配好的导出设置。
2. **改表写回**：`options.GetExportLayerTable()` 读出表 → 改条目（如 `ExportLayerInfo` 的 LayerName/Color）→ `options.SetExportLayerTable(table)` 写回 → 随 `doc.Export()` 生效。

好处：CAD 标准一旦表格化，就能批量套用到所有导出，不必在 UI 里逐项手配。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 按公司 CAD 标准批量配置图层映射
- **问题**: 公司标准要求所有 Revit 墙类别导出为 DWG 的 "WALL" 图层、颜色 31，手工在 UI 逐项设置不可维护。
- **方法论的使用**: 走"拿选项 → 改表 → 写回"：取（或新建）导出选项 → `GetExportLayerTable()` → 将墙类别对应条目改为 `ExportLayerInfo { LayerName = "WALL", ColorName = "31", CutColorNumber = 31 }` → `SetExportLayerTable` → `doc.Export()`。
- **结论**: 类别到图层的映射完全代码化，规则可入库复用。
- **结果**: 导出的 DWG 图层名与颜色符合标准，多项目批量应用无需人工干预。

### 案例 2: 复用文档中的预设导出设置
- **问题**: 项目已有在 UI 中调好的导出设置（setup），插件希望沿用同一套而不是重建。
- **方法论的使用**: `BaseExportOptions.GetPredefinedSetupNames(doc)` 枚举预设名 → `GetPredefinedOptions(doc, setupName)` 取到该预设的选项对象。
- **结论**: 预设即"已保存的表配置"，代码与 UI 共享同一数据源。
- **结果**: 插件导出结果与人工用该预设导出一致，避免双轨配置漂移。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 导出 DWG/DXF/DGN 必须满足企业 CAD 标准（图层名、颜色、线型、线宽、字体）。
2. 要按类别/子类别精确控制导出到哪个 CAD 图层（如墙→WALL，门→DOOR）。
3. 插件想沿用用户在 UI 里配置好的导出预设，而不是硬编码选项。
4. 图纸交付方审图退回，要求改映射批量重导。

### 语言信号 (用户的话里出现这些就应激活)

- "导出图层映射/类别对应图层" / "export layer table / category to layer mapping"
- "按公司 CAD 标准导出" / "CAD standards on export"
- "ExportLayerInfo / SetExportLayerTable / GetPredefinedOptions"
- "复用导出预设" / "reuse predefined export setup"

### 与相邻 skill 的区分

- 与 `revit-document-export-formats`：依赖其先选定 CAD 格式与重载，本 skill 再在其上配置五张映射表——格式选型在前、映射配置在后。
- 与 `revit-printable-view-check`：导出前的视图集合过滤依赖其 CanBePrinted 检查，组合成完整 CAD 导出前置链路。
- 与 `revit-ifc-exporter-registration`：前者只在内置导出的映射表层配置规则，后者替换整个导出实现——配置层 vs 接管层，介入深度对立。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **取得导出选项**
   - 先 `GetPredefinedSetupNames(doc)` 看有无可用预设；有则 `GetPredefinedOptions(doc, setupName)` 取选项，无则 new 对应格式的 ExportOptions（DWG/DXF/DGN 同族）。
   - 完成标准: 手上有配置基点（预设复用或全新选项），并已向用户说明采用哪条路线。
   - 判停条件: 若目标格式不是 DWG/DXF/DGN，转 `revit-document-export-formats` 的对应选项类，本 skill 的五张表不适用。

2. **读取并修改映射表**
   - 按需 `GetExportLayerTable()`（或线型/图案/字体/线宽表），把标准中的每条规则改入对应条目（类别 key → 新 ExportLayerInfo 等），改完 `SetExportXxxTable(table)` 写回。
   - 完成标准: 全部标准规则（图层名、颜色、剖切颜色、线宽……）已落入表中，无遗漏项。

3. **导出并验证**
   - 组装视图集合（仅 CanBePrinted 的视图），调用 `doc.Export(...)`；在 CAD 中检查图层名/颜色/线型是否符合标准。
   - 完成标准: 抽查关键类别的映射结果与标准一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 非 CAD 格式导出（IFC、gbXML、SAT、FBX 等）——没有这五张映射表，各有自己的选项体系。
- 还没决定导出格式/重载——先走 `revit-document-export-formats`。
- 需要改变导出**过程**本身（自定义写出）——走自定义导出器注册，映射表管不到过程。

### 作者在书中警告的失败模式

- 只改了图层表忘了"剖切颜色"（CutColorNumber）与投影颜色是两个字段，平面/立面导出颜色不一致。
- 拿预设名调 `GetPredefinedOptions` 时名字拼写不匹配（预设名大小写/全半角），拿到空结果仍继续导出。
- 改表后忘记 SetExportXxxTable 写回选项对象，导出仍用旧映射。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：新版本导出设置（如按子类别图层细化、DWG 版本选项、共享参数→图层属性等）有扩展，表结构与字段需核对当前 API。
- 书中只给方法签名与原则，未提供完整的"企业标准 → 表条目"批量导入样例，落地时需自建数据模型。

### 容易混淆的邻近方法论

- `revit-document-export-formats`（格式决策）——本 skill 是选定 CAD 格式后的配置层。
- `revit-printable-view-check`（视图可打印性）——同属导出前置链路，职责不同勿混。

---

## 相关 skills

- **revit-document-export-formats**（depends-on）：依赖其先选定 CAD 格式与重载，本 skill 再在其上配置五张映射表——格式选型在前、映射配置在后。
- **revit-printable-view-check**（composes-with）：导出前的视图集合过滤依赖其 CanBePrinted 检查，组合成完整 CAD 导出前置链路。
- **revit-ifc-exporter-registration**（contrasts-with）：前者只在内置导出的映射表层配置规则，后者替换整个导出实现——配置层 vs 接管层，介入深度对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
