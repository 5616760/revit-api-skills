---
name: revit-transmissiondata-offline
description: |
  需要在不启动 Revit、不打开文档的情况下批量读取或修改一批 .rvt 文件的链接路径与加载状态时调用（如服务器迁移后批量修复失效链接、批量卸载某链接）。Trigger：transmission data、batch fix links、offline load state、离线修改链接、批量修复链接路径、不打开文件改链接。不适用于：打开文档内的链接创建/几何操作、添加或删除引用本身。注意：只能改现有引用，不能增删引用。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.15.2 数据传输（约p415-418）
tags: [revit-api, transmissiondata, offline-batch, link-path, file-management]
related_skills:
  - slug: revit-nested-link-unload
    relation: depends-on
  - slug: revit-external-file-reference-utils
    relation: contrasts-with
---

# TransmissionData 离线加载控制

## R — 原文 (Reading)

> 因此，不必打开完整的 Revit 相关文件，就可用 TransmissionData 执行外部文件引用操作。ReadTransmissionData() 和 WriteTransmissionData() 方法可用于获取有关外部引用的信息，或更改信息。
>
> — 宦国胜, 第5章 5.15.2 数据传输（约p415-418）

---

## I — 方法论骨架 (Interpretation)

每个 .rvt 文件内部保存着一份 TransmissionData（数据传输记录）：它记着外部文件引用"上次保存时的状态"与"下次打开时被请求的状态"。因此可以把它当成一个**离线开关面板**——不启动 Revit、不打开文档，直接对文件读/写链接的路径与是否加载。

标准用法是一个五步循环：

1. `TransmissionData.ReadTransmissionData(location)` 读出该文件的引用数据（location 是 ModelPath）。
2. `GetAllExternalFileReferenceIds()` 列出全部顶层引用 ID。
3. `GetLastSavedReferenceData(refId)` 查看每个引用的当前路径/状态。
4. `SetDesiredReferenceData(refId, newPath, pathType, isLoaded)` 设定下次打开时的期望状态（改路径、改加载）。
5. 需要"传输"语义时设 `IsTransmitted = true`，最后 `WriteTransmissionData(location, transData)` 写回。

关键限制：它只能**修改已有引用**的路径与加载状态，不能添加或删除引用；且只含顶层链接（嵌套链接随父链接传递处理，见嵌套链接 skill）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 删除/清空导出文件中的外部链接
- **问题**: 导出的文件仍带着外部链接信息，需要清理使交付文件不含有效外部引用。
- **方法论的使用**: 用 TransmissionData 离线遍历导出文件的引用 ID，对每个引用调用 `SetDesiredReferenceData` 将加载状态置为未加载，再 `WriteTransmissionData` 写回。
- **结论**: 不必在 Revit 中逐个打开文件，批量完成引用状态清理。
- **结果**: 交付文件再打开时链接不再自动加载，符合交付要求。

### 案例 2: 服务器迁移后批量修复失效链接
- **问题**: IT 迁移 Revit Server，大量文件链接路径全部失效，逐个打开修复不可行。
- **方法论的使用**: 遍历文件列表，对每个 ModelPath 走五步循环，把旧路径替换为新服务器路径并设 PathType。
- **结论**: 路径修复只改"下次打开的期望状态"，无需打开文档本身。
- **结果**: 全部文件一次性修复，下次打开时链接按新路径加载成功。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 服务器/网盘迁移后，成百上千个 .rvt 的链接路径失效，需要批量重定向。
2. 交付或归档前，需要批量把某些链接设为"不加载"（如卸载所有 CAD 链接）。
3. 需要在不安装/不启动 Revit 的环境（如后台脚本）里查询文件的链接清单与状态。
4. 读取"上次保存时"的引用路径做审计或对比。

### 语言信号 (用户的话里出现这些就应激活)

- "不用打开文件就改链接" / "without opening the file / offline"
- "批量修复链接路径" / "batch fix broken links / repath links"
- "TransmissionData / ReadTransmissionData / WriteTransmissionData"
- "几百个文件链接全失效了" / "mass repath after server migration"

### 与相邻 skill 的区分

- 与 `revit-nested-link-unload`：依赖其"嵌套随父卸载"的拓扑规则——离线遍历只处理顶层链接，不必也不可对嵌套链接单独操作。
- 与 `revit-external-file-reference-utils`：前者读未打开文件的内嵌传输数据，后者查已打开文档内的引用容器——一个离线一个在线，数据来源对立。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **读出传输数据**
   - 构造目标文件的 ModelPath，调用 `TransmissionData.ReadTransmissionData(location)`。
   - 完成标准: 拿到 TransmissionData 对象；若返回为空，确认文件无引用数据并报告。
   - 判停条件: 若目标只是"查询"，执行第 2 步后即可输出清单结束。

2. **枚举并读取引用现状**
   - `GetAllExternalFileReferenceIds()` → 逐个 `GetLastSavedReferenceData(refId)` 取路径与状态。
   - 完成标准: 得到"引用 ID → 当前路径/加载状态"清单。

3. **设定期望状态并写回**
   - 对需要修改的引用调用 `SetDesiredReferenceData(refId, newPath, pathType, isLoaded)`；如需传输语义设 `IsTransmitted = true`；最后 `WriteTransmissionData(location, transData)`。
   - 完成标准: 写回成功；注意**只改已有引用**，用户若要求"新增/删除链接"则停下说明此 API 做不到，需打开文档用链接 API。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要新增或删除一个链接引用——TransmissionData 只能改现有引用的路径与加载状态，做不到增删。
- 文档已经打开、可以交互操作时——直接用打开文档的链接 API 更直接可靠。
- 需要操作链接内部几何或图元——离线数据不含几何信息。

### 作者在书中警告的失败模式

- 忘记 WriteTransmissionData 写回，或写回了但没设 IsTransmitted，导致期望状态未生效。
- 试图遍历并单独处理嵌套链接——TransmissionData 只含**顶层**链接，嵌套链接随父链接自动处理（见 revit-nested-link-unload）。
- 修改了路径但 PathType 选错（如服务器路径仍设 Relative），下次打开仍找不到文件。

### 作者的盲点 / 时代局限

- 基于 Revit 2014，未覆盖后来的云端模型（BIM 360/ACC）路径形态；云路径的离线重定向需要核对新版 API。
- 书中未讨论文件被他人占用/权限失败时的并发写入问题。

### 容易混淆的邻近方法论

- `revit-external-file-reference-utils`（引用查询容器/工具类）——本 skill 是"文件级离线状态读写"。
- `revit-link-decision-flow`（在线链接操作）——打开文档时不要用离线方式绕路。

---

## 相关 skills

- **revit-nested-link-unload**（depends-on）：依赖其"嵌套随父卸载"的拓扑规则——离线遍历只处理顶层链接，不必也不可对嵌套链接单独操作。
- **revit-external-file-reference-utils**（contrasts-with）：前者读未打开文件的内嵌传输数据，后者查已打开文档内的引用容器——一个离线一个在线，数据来源对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
