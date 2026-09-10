---
name: revit-document-function-map
description: |
  把 Document 当作功能域聚合体导航 API：项目信息/位置、类型集合、视图、图元检索/管理、文件、事件、设置（Settings→Materials/Units）。找入口先问"属于哪个功能域"；读单位用 GetUnits()，材质走 Settings.Materials。信号："Document 有什么功能 / what does Document provide"、"在哪读单位/材质 / where to read units materials"。不适用：具体过滤/编辑/参数细节。
  Trigger: Document class / Settings / GetUnits / document API map。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.4.2（约p059-060）
tags: [revit-api, document, api-map, settings, type-collections]
related_skills:
  - slug: revit-element-retrieval-four-entries
    relation: composes-with
  - slug: revit-document-vs-uidocument
    relation: depends-on
---

# Document 核心功能地图

## R — 原文 (Reading)

> Document 类主要提供以下功能：设置属性（Settings Property）、地点和位置（Place and Locations）、类型集合（Type Collections）、视图管理（View Management）、图元检索（Element Retrieval）……
>
> — 宦国胜, 第1章 1.4.2（约p059-060）

---

## I — 方法论骨架 (Interpretation)

Document 是所有 Revit API 操作的中心对象，但它不是一个单一职责的类，而是九大功能域的聚合：

1. **设置**：Settings 属性 → Materials（材质）、Units（单位）等项目级设置。
2. **地点与位置**：项目基地点、正北等地理位置信息。
3. **类型集合**：FloorTypes、WallTypes 等族类型集合，取类型对象从这里走。
4. **视图管理**：视图的创建与访问。
5. **图元检索**：GetElement、FilteredElementCollector 的宿主。
6. **文件管理**：Open/Save/Close 及文件事件。
7. **图元管理**：图元的创建、删除、修改。
8. **事件**：文档生命周期事件。
9. **文件状态**：文档当前状态（是否只读、是否修改过等）。

用法是"先定域再找入口"：任何以 Document 为起点的需求，先判断它属于哪个功能域，再进对应属性/方法。例如读默认长度单位属于"设置"域 → `Document.GetUnits()`；拿某类墙的类型属于"类型集合"域 → 文档的类型集合属性。后续所有过滤、编辑、参数访问都以这张地图为坐标系。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 读取项目单位
- **问题**: 需要读取项目的默认长度单位。
- **方法论的使用**: 定位到"设置"功能域 → `Document.GetUnits()` 返回 Units 对象，从中读长度单位格式。
- **结论**: 单位不挂在某个图元上，挂在 Document 的设置域。
- **结果**: 不经过任何图元遍历就拿到项目单位配置。

### 案例 2: Settings 属性映射
- **问题**: 第1章 1.4.4 需要访问项目级设置对象（如材质）。
- **方法论的使用**: 走 Document.Settings 属性进入设置域，再导航到 Materials 等子集合。
- **结论**: "Document.Settings.XXX"是项目级配置的标准路径。
- **结果**: 材质等全局资源经统一入口访问。

### 案例 3: 类型集合的使用
- **问题**: 创建墙/楼板前要先拿到对应类型对象。
- **方法论的使用**: 走"类型集合"功能域的 FloorTypes/WallTypes 等属性。
- **结论**: 找类型不靠全文档过滤，靠文档类型集合属性直达。
- **结果**: 类型获取路径短且稳定。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 刚开始写 Revit 代码，不知道某个能力（单位、材质、位置、类型）从哪个对象上取。
2. 在图元/过滤器里翻找项目级资源（材质、单位），方向错了。
3. 需要判断某操作（保存、事件注册、状态查询）是不是 Document 的职责。

### 语言信号 (用户的话里出现这些就应激活)

- "Document 能干什么 / what does Document provide / Document properties"
- "怎么读单位/材质 / read units / access materials"
- "项目信息从哪拿 / project information / project location"
- "类型集合 / wall types floor types / type collections"

### 与相邻 skill 的区分

- 与 `revit-document-vs-uidocument` 的区别: 那个讲 Document 与 UIDocument 的 UI/DB 分工边界；本 skill 讲 Document 内部九域怎么导航。先分清对象层级，再进本地图。
- 与 `revit-element-retrieval-four-entries` 的区别: 检索四法是"图元检索"功能域内部的展开；本地图告诉你什么时候根本不该进检索域（比如取类型直接走类型集合）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **把需求归类到功能域**：设置/位置/类型集合/视图/检索/文件/图元管理/事件/状态，九选一。
   - 完成标准: 写下需求所属功能域。

2. **沿域找入口**：进入对应属性或方法（如 Settings→Materials、GetUnits()、WallTypes），取到目标对象。
   - 完成标准: 代码入口与功能域对应，无跨域绕路（如过滤全文档找类型）。
   - 判停条件: 若需求是"按条件找一批图元"→ 转入图元检索域，交给过滤 skill 处理。

3. **确认无更短路径**：检查是否有 Document 直达属性可替代手工遍历。
   - 完成标准: 所用路径是地图上该域的标准入口。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已经知道入口，讨论的是具体操作实现（过滤组合、移动图元、参数读写）——转对应执行类 skill。
- 需要的是 UI 层操作（选择、提示框）——那是 UIDocument/Application 层的事。

### 作者在书中警告的失败模式

- 在图元层或过滤器里找项目级资源（单位、材质、类型集合）——方向性错误，正确入口在 Document 的对应功能域。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：九域框架仍然成立，但新版 Document 增补了不少便捷属性与方法（如更多 GetXxx 快捷访问），实际地图比书中列举略丰富。

### 容易混淆的邻近方法论

- Document 的"类型集合"域与 FamilySymbol 过滤都能拿类型对象：前者是直达入口（首选），后者是检索路径（兜底）——优先级不要颠倒。

---

## 相关 skills

- revit-element-retrieval-four-entries：composes-with——本地图把"图元检索"功能域指给四法决策，四法是该域内部的展开。
- revit-document-vs-uidocument：depends-on——本 skill 讲 Document 内部九域导航，前提是已分清 Document 与 UIDocument 的层级边界。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
