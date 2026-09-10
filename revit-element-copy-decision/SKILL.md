---
name: revit-element-copy-decision
description: |
  复制图元按目标域四选一：同文档 CopyElement/CopyElements、跨文档 CopyElements、跨视图 CopyElements（仅视图专用图元）。跨文档重名冲突默认弹模态框，自动化需 CopyPasteOptions+IDuplicateTypeNamesHandler；CopyElement 返回 ElementId 集合。信号："复制图元 / copy element"、"类型重名 / duplicate type names"。
  Trigger: CopyElement / CopyElements / CopyPasteOptions。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.5.2 / 表 2-8（约p091）
tags: [revit-api, copy, elementtransformutils, copy-paste, cross-document]
related_skills:
  - slug: revit-elementid-vs-uniqueid
    relation: composes-with
  - slug: revit-element-move-decision
    relation: contrasts-with
---

# 图元复制决策树：CopyElement / CopyElements / 跨文档复制 / CopyPasteOptions

## R — 原文 (Reading)

> CopyElement(Document, ElementId, XYZ) 复制图元并将其置于给定转换所指定的位置。
> CopyElements(Document, ICollection<ElementId>, XYZ) 复制一组图元并将其置于给定转换所指定的位置。
>
> — 宦国胜, 第2章 2.5.2 / 表 2-8（约p091）

---

## I — 方法论骨架 (Interpretation)

复制 API 按"目标在哪"分四种，选错会直接失败或达不到意图：

1. **同文档单图元**：`CopyElement(doc, id, translation)` —— 复制一个图元并按偏移量放置。注意返回值是 **ElementId 集合**（不是单个 ID），因为复制可能连带产生关联图元。
2. **同文档批量**：`CopyElements(doc, idCollection, translation)` —— 一组图元统一偏移复制。
3. **跨文档**：`CopyElements(srcDoc, ids, dstDoc, transform, options)` —— 从源文档（含链接文档）复制到目标文档，Transform 处理坐标系差异。类型重名冲突默认弹**模态对话框**打断自动化——要静默处理需传 `CopyPasteOptions` 并实现 `IDuplicateTypeNamesHandler`。
4. **跨视图**：`CopyElements(srcView, ids, dstView, transform, options)` —— 只对**视图专用图元**（注释、详图等）有效；模型图元不属于视图，走这条路径没意义，应走文档级复制。

设计上复制与移动是平行的变换族：同一个"要不要变目标/要不要保留原件"的问法。跨文档复制是族实例等对象跨文件传递的基础路径。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 同文档偏移复制
- **问题**: 在当前文档复制一个/一组图元并平移放置。
- **方法论的使用**: 表 2-8 前两行——CopyElement/CopyElements + XYZ 偏移量。
- **结论**: 同文档复制按数量选单/批量入口。
- **结果**: 副本按偏移落位，原件保留。

### 案例 2: 链接文档复制与重名自动化
- **问题**: 从链接文档复制图元到当前文档，遇到重名类型冲突，默认弹模态框打断批处理。
- **方法论的使用**: 用跨文档 CopyElements + CopyPasteOptions，实现 IDuplicateTypeNamesHandler 自动决策重名（如统一用目标文档版本）。
- **结论**: 冲突处理策略可以也应该代码化，别依赖对话框。
- **结果**: 跨文档复制无人值守完成。

### 案例 3: 族实例跨文档创建
- **问题**: 需要在当前文档创建来自其他文档的族实例。
- **方法论的使用**: 走跨文档复制逻辑把族实例（及其类型）带入目标文档。
- **结论**: 跨文档复制是对象跨文件传递的基础路径。
- **结果**: 族实例连同依赖类型一起到位。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写"阵列/备份/克隆"类命令，要复制单个或一批图元。
2. 从链接模型（Linked RVT）把图元带到当前文档。
3. 跨文档复制时程序卡在模态对话框——重名类型冲突没自动化。
4. 想把注释从一张视图抄到另一张视图（跨视图复制的正用例），或误用它复制模型图元。

### 语言信号 (用户的话里出现这些就应激活)

- "复制图元 / copy element / duplicate elements"
- "从链接文档复制 / copy from linked model / cross-document copy"
- "类型重名冲突 / duplicate type names handler"
- "复制到另一个视图 / copy to another view"

### 与相邻 skill 的区分

- 与 `revit-element-move-decision` 的区别: 移动不保留原件、向量语义相同；复制保留原件且返回新 ID——需求里"要不要保留原件"是分岔点。
- 与 `revit-filter-quick-slow-logical` 等检索类 skill 的区别: 本 skill 处理"拿到图元之后"的变换；检索是上游。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定复制目标域**：同文档 / 跨文档 / 跨视图，三选一；确认"要保留原件"。
   - 完成标准: 目标域明确；跨视图路径仅当选中的是视图专用图元（注释类）。
   - 判停条件: 若是模型图元想"复制到另一视图"——语义不成立，转文档级复制（模型图元本来就对所有视图可见）。

2. **选方法与参数**：单个 → CopyElement；批量 → CopyElements；跨文档 → srcDoc/dstDoc/Transform/CopyPasteOptions；跨视图 → srcView/dstView 版本。
   - 完成标准: 方法签名与目标域匹配，Transform/偏移量就绪。

3. **处理重名冲突（跨文档/跨视图）**：实现 IDuplicateTypeNamesHandler 塞进 CopyPasteOptions，避免模态对话框。
   - 完成标准: 跨域复制路径中无人工交互依赖。

4. **接收返回 ID 集合**：CopyElement 也返回 ElementId 集合，按集合处理结果，不要按单 ID 取下标。
   - 完成标准: 返回值按集合接收并校验数量。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需求是移动（原件不保留）——转移动决策树。
- 跨视图复制模型图元——模型图元不属于任何视图，该路径只对视图专用图元有效。

### 作者在书中警告的失败模式

- 跨文档复制不配 IDuplicateTypeNamesHandler——重名时弹模态框，自动化流程被打断或卡死。
- 把 CopyElement 返回值当单个 ElementId 用——实际是集合，可能含关联图元。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版复制 API 基本稳定，但链接文档的访问方式（RevitLinkOptions/链接实例导航）有演进，跨文档复制的细节行为建议对照新版文档验证。

### 容易混淆的邻近方法论

- "复制"与"新建同类型实例"都能产出类似对象：复制保留原参数与关联；新建实例要重新赋参——语义不同，别拿复制替代批量创建，也别拿创建替代定向复制。

---

## 相关 skills

- revit-elementid-vs-uniqueid：composes-with——复制返回新 ElementId 集合，跨文档/跨项目跟踪副本时需转用 UniqueId。
- revit-element-move-decision：contrasts-with——复制保留原件并返回新 ID，移动不保留原件，两者 API 家族相邻但语义相反。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
