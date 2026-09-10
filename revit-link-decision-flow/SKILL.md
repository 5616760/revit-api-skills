---
name: revit-link-decision-flow
description: |
  需要以编程方式判断图元是否引用外部文件、创建/加载/卸载 Revit 链接、或配置链接的路径类型（Relative/Absolute/Server）与附着类型（Attachment/Overlay）时调用。Trigger：link file、external reference、load/unload link、加载链接、卸载链接、嵌套链接、覆盖模式。不适用于：纯几何交互（改用 Reference 转换 skill）、批量离线改路径（改用 TransmissionData skill）、CAD/DWF 等导出（改用 Export 决策 skill）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.15 链接文件（约p410-415）
tags: [revit-api, linked-file, revit-link, external-reference, decision-flow]
related_skills:
  - slug: revit-transmissiondata-offline
    relation: contrasts-with
  - slug: revit-link-reference-conversion
    relation: composes-with
  - slug: revit-external-file-reference-utils
    relation: composes-with
  - slug: revit-nested-link-unload
    relation: composes-with
---

# 链接文件决策流程

## R — 原文 (Reading)

> 包含外部文件引用的图元是这样一个图元，它引用"基础".rvt 文件以外的某些文件。判断图元有无外部文件，请使用 Element.IsExternalFileReference()。Element.GetExternalFileReference() 返回 ExternalFileReference。
>
> — 宦国胜, 第5章 5.15 链接文件（约p410-415）

---

## I — 方法论骨架 (Interpretation)

处理"链接文件"问题不要一步到位写操作代码，而是走一条八步决策链，先识别、再分类、后操作：

1. **识别**：对目标图元调用 `IsExternalFileReference()`，先判断"是不是外部引用"，避免对普通图元做无意义操作。
2. **获取**：用 `GetExternalFileReference()` 拿到引用对象，里面含路径与类型信息。
3. **分类**：判断引用属于 Revit 链接、CAD 链接还是贴花等——不同类型的后续 API 完全不同。
4. **创建**：新建 Revit 链接是两步——先 `RevitLinkType.Create()` 建类型，再 `RevitLinkInstance.Create()` 建实例。
5. **加载状态**：用 Load/Unload 控制与 `IsLoaded` 查询。
6. **嵌套关系**：通过父链接的 `GetChildIds()` 逐级向下遍历。
7. **路径类型**：PathType 三选一（Relative/Absolute/Server）。
8. **附着类型**：AttachmentType 二选一（Attachment/Overlay），决定父链接被直接打开时子链接是否可见。

核心思想：每一步的输出决定下一步走哪个分支，跳步（比如不判断类型直接创建）是链接操作出错的主要来源。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 以 Overlay 模式编程加载一个 Revit 链接
- **问题**: 需要让新加载的链接在父链接被直接打开时仍唯一可见，且不能影响链接树的传递行为。
- **方法论的使用**: 按决策链先建类型再建实例：`RevitLinkType.Create(doc, modelPath, options)` 时在 options 中设 `AttachmentType = Overlay`，随后 `RevitLinkInstance.Create(doc, linkTypeId)` 建实例。
- **结论**: 两步创建成功后，用 `RevitLinkLoadResults.LoadResult == RevitLinkLoadResultType.LinkLoaded` 验证加载结果。
- **结果**: 链接以覆盖模式加载成功；直接打开父链接时只有 Overlay 链接可见，与 Attachment 行为区分明确。

### 案例 2: 识别文件中的全部外部引用并分类
- **问题**: 需要列出当前文档引用了哪些外部文件，并区分 Revit 链接与 CAD 链接分别处理。
- **方法论的使用**: 从第 1-3 步入手——遍历图元用 `IsExternalFileReference()` 过滤，再 `GetExternalFileReference()` 取引用，按类型分类路由到不同处理分支。
- **结论**: 得到按类型分组的引用清单，Revit 链接走类型/实例 API，CAD 链接走各自的导入设置。
- **结果**: 后续的加载、卸载、路径修改都能按正确的 API 族执行，不再混用。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写插件时要以编程方式加载、卸载或重建一个 Revit 链接（含设置相对/绝对/服务器路径）。
2. 想让某个链接以 Overlay 模式加载，使父链接被直接打开时其他子链接不可见。
3. 需要遍历文档中所有链接并按 Revit/CAD/贴花分类做不同处理。
4. 需要检查或调整链接树的父子（嵌套）关系。

### 语言信号 (用户的话里出现这些就应激活)

- "加载/卸载/重载一个链接" / "load / unload / reload a linked Revit file"
- "覆盖模式还是附着模式" / "Overlay vs Attachment"
- "相对路径还是绝对路径，服务器路径" / "relative / absolute / server path"
- "怎么用代码创建一个 Revit 链接" / "create RevitLinkType / RevitLinkInstance programmatically"

### 与相邻 skill 的区分

- 与 `revit-transmissiondata-offline`：前者面向打开文档的在线创建/加载/卸载链接，后者面向未打开文件的离线批量读写路径与加载状态——操作场景与数据来源对立。
- 与 `revit-link-reference-conversion`：前者管链接文件层的创建/加载/分类，后者管链接内几何的拾取与 Reference 转换——文件层之上接几何参照层，组合成完整链接工作流。
- 与 `revit-external-file-reference-utils`：前者的识别与分类步骤依赖后者的 IsExternalFileReference/GetAllExternalFileReferences 查询工具——查询在前、完整决策在后。
- 与 `revit-nested-link-unload`：前者的嵌套遍历与卸载操作须遵循后者的传递性约束（卸载在父链接节点解决）——决策链组合其拓扑规则。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **识别与获取引用**
   - 对目标图元调用 `Element.IsExternalFileReference()`；为 true 时调用 `GetExternalFileReference()` 取得引用。
   - 完成标准: 已明确目标是不是外部引用，并拿到 ExternalFileReference 对象（或确认全部候选都不是）。
   - 判停条件: 若无任何图元是外部引用，直接报告"无链接"并结束。

2. **分类并路由**
   - 判断引用类型（Revit 链接 / CAD 链接 / 贴花等），Revit 链接继续第 3 步。
   - 完成标准: 每个引用已归入确定类别并选定对应 API 族。
   - 判停条件: 若只有 CAD 链接，转到 CAD 导入/导出相关逻辑，不走 RevitLinkType API。

3. **执行创建/加载操作**
   - 新建链接：`RevitLinkType.Create(doc, modelPath, options)`（设置 PathType 与 AttachmentType）→ `RevitLinkInstance.Create(doc, linkTypeId)`。
   - 完成标准: 检查 `RevitLinkLoadResults.LoadResult == LinkLoaded`，类型与实例均已存在且加载成功。

4. **处理嵌套与验证状态**
   - 用 `GetChildIds()` 遍历嵌套链接（如需要），用 `IsLoaded` 验证最终加载状态。
   - 完成标准: 链接树状态与需求一致（Overlay/Attachment、加载/卸载）并可向用户报告。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要在**不打开**文档的情况下批量修改链接路径/加载状态——那是 TransmissionData 的职责，本 skill 的 API 都要求文档已打开。
- 只想枚举"哪些图元有外部引用"的查询需求——`ExternalFileUtils` 一步即可，无需走完整决策链。
- 修改链接内几何或拾取链接内图元——走 Reference 转换（`CreateLinkReference`）。

### 作者在书中警告的失败模式

- 跳过类型判断直接操作，把 CAD 链接当 Revit 链接处理，API 族用错导致异常。
- 混淆 Overlay 与 Attachment：直接打开父链接时只有 Overlay 链接可见，配错会导致链接意外可见/不可见。
- 忘记检查 `RevitLinkLoadResults.LoadResult`，加载失败的链接被当作成功继续后续流程。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版 Revit 的链接 API（如 RevitLinkOptions、加载后重新定位、BIM 360/云端路径）与 2014 有差异，套用旧签名需核对当前版本。
- PathType 的 Server 指向旧 Revit Server，对后来的云协作工作流覆盖不足。

### 容易混淆的邻近方法论

- `revit-transmissiondata-offline`（离线批量改引用）——本 skill 是"打开文档在线操作"。
- `revit-nested-link-unload`（嵌套链接传递卸载）——嵌套遍历时不需（也不能）单独卸载子链接。

---

## 相关 skills

- **revit-transmissiondata-offline**（contrasts-with）：前者面向打开文档的在线创建/加载/卸载链接，后者面向未打开文件的离线批量读写路径与加载状态——操作场景与数据来源对立。
- **revit-link-reference-conversion**（composes-with）：前者管链接文件层的创建/加载/分类，后者管链接内几何的拾取与 Reference 转换——文件层之上接几何参照层，组合成完整链接工作流。
- **revit-external-file-reference-utils**（composes-with）：前者的识别与分类步骤依赖后者的 IsExternalFileReference/GetAllExternalFileReferences 查询工具——查询在前、完整决策在后。
- **revit-nested-link-unload**（composes-with）：前者的嵌套遍历与卸载操作须遵循后者的传递性约束（卸载在父链接节点解决）——决策链组合其拓扑规则。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
