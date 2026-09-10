---
name: revit-document-export-formats
description: |
  需要把 Revit 文件（或部分图元/视图）导出为其他格式时调用，先按 13 种格式（gbXML/IFC/NWC/DWF/DWFX/FBX/DGN/DWG/DXF/SAT/ADSK 等）选对 Document.Export 重载与对应 ExportOptions。Trigger：export to DWG/IFC/Navisworks/gbXML、导出为某格式、Document.Export、ExportOptions。注意：Navisworks 只能以插件导出器导出；gbXML 分质量模型与绿色建筑两种选项；导出前视图必须 CanBePrinted。不适用于：导入、打印（非导出 API）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.16 导出（约p418）
tags: [revit-api, export, document-export, format-decision, file-io]
related_skills:
  - slug: revit-printable-view-check
    relation: depends-on
  - slug: revit-ifc-exporter-registration
    relation: contrasts-with
---

# Document.Export 多格式决策

## R — 原文 (Reading)

> Revit API 允许对 Revit 文件或其一部分，导出到其他软件使用的各种格式。Document 类有一个可重载的 Export() 方法，可使用 Revit 内置导出程序启动文件导出。
>
> — 宦国胜, 第5章 5.16 导出（约p418）

---

## I — 方法论骨架 (Interpretation)

导出的入口只有一个：`Document.Export()` 的**重载家族**——每种格式一个重载，各自要求不同的"选项对象 + 视图集合 + 目标路径"组合。所以导出开发的第一件事不是写代码，而是做**格式决策**：

1. **定格式**：gbXML、IFC、NWC（Navisworks）、DWF/DWFX、FBX、DGN、DWG、DXF、SAT、ADSK 等，按下游软件需求选。
2. **选选项类**：每种格式有自己的 ExportOptions；同族格式有共享基类（DWG/DXF/DGN 共用 BaseExportOptions 体系）。
3. **辨易混分支**：
   - gbXML 分两种——`GBXMLExportOptions`（绿色建筑模型）与 `MassGBXMLExportOptions`（仅概念体量族文件）。
   - Navisworks **只能**通过插件导出器导出，没有普通重载可用。
   - DWFX 本质是 XPS 封装格式，与 DWF 选项相近但签名不同。
4. **查前置条件**：导出所用的每个视图必须 `CanBePrinted == true`，否则静默失败。
5. **定制需求**：IFC 与 Navisworks 的导出过程可通过注册自定义导出器接管（IExporterIFC 等）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: gbXML 导出选项的选择
- **问题**: 项目要做能耗分析需要导出 gbXML，但项目同时含概念体量族和普通建筑图元，不知道选哪个选项类。
- **方法论的使用**: 按格式决策辨分支——`GBXMLExportOptions` 面向绿色建筑模型，`MassGBXMLExportOptions` 仅适用于概念体量族文件。
- **结论**: 混合项目以绿色建筑模型为主时选 GBXMLExportOptions，因为体量导出只关注体量分析。
- **结果**: 导出的 gbXML 覆盖普通图元的能耗语义，满足分析软件要求。

### 案例 2: 指定视图集合导出 DWG
- **问题**: 只要把当前视图导出为 DWG 交付。
- **方法论的使用**: 选 DWG 重载（路径 + DWGExportOptions + 视图集合），导出前检查每个视图 `CanBePrinted`。
- **结论**: 视图有效性与选项配置齐备后调用 Export 即可。
- **结果**: DWG 生成成功；不可打印视图（模板/隐藏视图）被预先过滤掉，未出现"无文件无报错"的静默失败。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写导出插件：把当前模型/选定视图导出为 DWG、DXF、DGN、SAT、IFC、gbXML 等。
2. 用户问"为什么没有 Navisworks 导出的重载/选项类"。
3. 能耗/绿建分析流程需要 gbXML，纠结两个 gbXML 选项类的区别。
4. 导出后没生成文件也没报错，排查前置条件。

### 语言信号 (用户的话里出现这些就应激活)

- "导出为 DWG/IFC/SAT/gbXML" / "export to DWG / IFC / SAT / gbXML"
- "Document.Export 怎么用" / "which Export overload"
- "Navisworks 导出" / "Navisworks export / NWC exporter"
- "导出没生成文件" / "export produced no file"

### 与相邻 skill 的区分

- 与 `revit-printable-view-check`：导出决策链的前置校验依赖其 CanBePrinted 红线检查——导出前先过滤不可打印视图，避免静默失败。
- 与 `revit-ifc-exporter-registration`：前者用内置导出程序选格式与重载，后者替换导出实现本身——默认 vs 接管，两条对立的导出路线。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定格式与重载**
   - 与用户确认目标格式；查该格式的 Document.Export 重载签名（参数顺序：目标文件夹/文件名、选项对象、视图集合，个别格式如 SAT/IFC 签名不同）。
   - 完成标准: 格式、重载与选项类三者匹配（Navisworks 除外——转插件导出器路线）。
   - 判停条件: 若格式是 NWC，说明无普通重载，转到自定义/插件导出器方案。

2. **配置选项对象**
   - 实例化对应 ExportOptions；gbXML 按内容选 GBXMLExportOptions 或 MassGBXMLExportOptions；DWG/DXF/DGN 需映射表配置时转 `revit-dwg-export-tables`。
   - 完成标准: 选项对象已按需求配置完毕。

3. **校验视图并执行导出**
   - 过滤视图集合，仅保留 `CanBePrinted == true` 的视图；调用 `doc.Export(...)`；检查返回值与产物文件。
   - 完成标准: 导出目录出现预期文件；若仍无文件，回头检查路径写权限与视图集合非空。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需求是"导入"外部文件进 Revit——导入 API 与导出体系完全不同。
- 需求是批量出图打印（Plot）而非格式交换——走打印/PrintManager 相关 API。
- 想深度接管 IFC 写出过程——本 skill 只用内置导出程序，接管走 `revit-ifc-exporter-registration`。

### 作者在书中警告的失败模式

- Navisworks：试图找普通 Export 重载导出 NWC——不存在，必须作为插件导出程序实现。
- gbXML：给含普通图元的项目误用 MassGBXMLExportOptions——体量导出只关注体量分析，普通图元语义丢失。
- 视图不可打印却未过滤——导出静默失败，无文件也无异常（详见 `revit-printable-view-check`）。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：13 种格式的清单与签名不完整覆盖新版本（如后来的 PDF 导出、IFC4 选项、点云/3D 形状导出等），迁移时需核对当前 API。
- 书中对各格式选项参数的枚举值覆盖不全，个别默认行为需实测。

### 容易混淆的邻近方法论

- `revit-dwg-export-tables`（CAD 映射表配置）——选完 DWG/DXF/DGN 后的下一步。
- `revit-printable-view-check`（可打印性红线）——导出前必查。
- `revit-ifc-exporter-registration`（自定义导出器）——"更进一步需求"的另一条路。

---

## 相关 skills

- **revit-printable-view-check**（depends-on）：导出决策链的前置校验依赖其 CanBePrinted 红线检查——导出前先过滤不可打印视图，避免静默失败。
- **revit-ifc-exporter-registration**（contrasts-with）：前者用内置导出程序选格式与重载，后者替换导出实现本身——默认 vs 接管，两条对立的导出路线。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
