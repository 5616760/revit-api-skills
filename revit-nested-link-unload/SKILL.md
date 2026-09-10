---
name: revit-nested-link-unload
description: |
  处理多级嵌套链接（A→B→C）的加载/卸载时调用，避免试图单独遍历并卸载嵌套链接。核心事实：TransmissionData 与卸载操作只处理顶层链接，嵌套链接随父链接卸载而自动卸载。Trigger：nested link、unload child link、嵌套链接、子链接卸载、多级链接树、GetChildIds。反信号：试图在 TransmissionData 里找嵌套链接 ID——它根本不含。不适用于：无嵌套的单层链接、链接几何交互。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.15.2 数据传输 代码5-60注释（约p415）
tags: [revit-api, nested-link, unload, link-topology, counter-example]
related_skills: []
---

# 嵌套链接卸载通过父链接实现

## R — 原文 (Reading)

> /// However, nested links will be unloaded if their parent links are unloaded, so this function only needs to look at the document's immediate links.
>
> — 宦国胜, 第5章 5.15.2 数据传输 代码5-60 注释（约p415）

---

## I — 方法论骨架 (Interpretation)

这是一个反直觉的拓扑规则：**嵌套链接不是独立的管理对象，而是父链接的附属**。

- 卸载传递性：卸载一个父链接，它的全部嵌套（子、孙）链接随之自动卸载；反之，重新加载父链接后子链接按其自身状态恢复。
- 处理范围因此收缩：任何针对"这个文档的链接"的批量操作（包括 TransmissionData 离线卸载），只需看**直接（顶层）链接**，不必递归到链接树深处。
- 常见错误方向：开发者想把链接树递归展开，对每个嵌套链接 ID 单独设置卸载——但 TransmissionData 根本不包含嵌套链接信息，这条路径既做不到也没必要。
- 如果只是想**查看**嵌套链接结构（而非卸载），才用父链接的 `GetChildIds()` 逐级下钻——那是遍历工具，不是操作入口。

一句话：卸载需求在父节点解决，遍历需求才下钻。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: TransmissionData 批量卸载函数只看直接链接
- **问题**: 需要写一个函数卸载文档的所有指定链接，文档中存在 A→B→C 多级嵌套，担心漏掉深层链接。
- **方法论的使用**: 书中代码 5-60 的注释点明：嵌套链接会随父链接卸载而卸载，函数只需遍历文档的**直接**链接。
- **结论**: 遍历 `GetAllExternalFileReferenceIds()` 的顶层结果即可，对每个顶层链接 SetDesiredReferenceData(..., isLoaded=false)。
- **结果**: 卸载 C 级链接的需求通过卸载其顶层祖先 B/A 自动达成，代码无需递归，也没有"找不到嵌套链接 ID"的异常。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写批量卸载/重载脚本时遇到多级嵌套链接，纠结"要不要递归遍历整棵链接树"。
2. 用户在 TransmissionData 的引用 ID 集合里找不到嵌套链接，以为数据丢了或 API 有 bug。
3. A→B→C 三级链接，想知道"卸载 B 之后 C 会怎样"。
4. 需要只卸载某个深层子链接而不动其父链——先来确认可行性。

### 语言信号 (用户的话里出现这些就应激活)

- "嵌套链接怎么卸载" / "unload nested link / child link"
- "链接树要递归吗" / "do I need to recurse the link tree"
- "TransmissionData 里找不到子链接" / "nested link not in transmission data"
- "卸载父链接子链接会怎样" / "what happens to nested links when parent unloads"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **画出顶层链接清单**
   - 用 `GetAllExternalFileReferenceIds()`（或打开文档时的顶层链接枚举）列出**直接**链接，不要递归展开。
   - 完成标准: 得到仅含顶层的链接清单，且已向用户说明嵌套不在此列。

2. **在父节点执行卸载**
   - 对需要移除的链接分支，找到其**顶层祖先**链接并对其执行卸载（在线用 Unload，离线用 SetDesiredReferenceData(..., isLoaded=false)）。
   - 完成标准: 只对顶层链接调用了卸载操作；未出现对嵌套链接 ID 的写操作。
   - 判停条件: 若用户要求"只卸载某个深层子链接、父链接保持加载"，停下说明此模型下做不到（需打开链接文件在其内部处理），给出替代方案。

3. **验证传递效果**
   - 卸载后检查父链接 IsLoaded 为 false，并确认其子链接（如需查看用 GetChildIds）不再出现在已加载集合中。
   - 完成标准: 整条分支按预期整体卸载。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单层链接（无嵌套）——直接对链接本体操作即可，无需拓扑分析。
- 只想**枚举/查看**嵌套链接结构——用 GetChildIds 遍历即可，本 skill 讲的是卸载语义不是遍历方法。
- 需要修改嵌套链接内部图元——打开那个链接文件本身，与卸载传递无关。

### 作者在书中警告的失败模式

- 试图在 TransmissionData 中查找并单独设置嵌套链接——数据中根本不存在嵌套链接条目，遍历逻辑必然落空。
- 递归遍历链接树并对每层调用卸载——多余的代码，且子链接单独卸载的调用往往无效或异常。
- 误以为父链接卸载后子链接仍"独立存活"，后续代码去取不存在的子链接几何而失败。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：新版 Revit 对嵌套链接的可见性/加载控制（如按需加载部分链接、链接管理器 UI）能力有变化，个别"做不到"的结论需要按当前版本复核。
- 书中未展开"重新加载父链接后子链接状态恢复"的完整细节，需实测确认。

### 容易混淆的邻近方法论

- `revit-transmissiondata-offline`（离线批量流程）——本 skill 是它的嵌套处理规则。
- `revit-external-file-reference-utils`（只含顶层的查询语义）——查询侧的同源约束，勿混淆"看"与"卸"。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
