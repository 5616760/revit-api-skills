---
name: revit-document-vs-uidocument
description: |
  Revit API 的 UI 层与 DB 层严格分离：界面操作（选择、提示、视图激活）必须经 UIDocument，模型/数据操作经 Document。无 UI 语境（IExternalDBApplication、后台批处理）只能经 Application.Documents 用 DB 层。选错层级会编译失败或运行异常。信号："UIDocument 和 Document 区别 / uidocument vs document"、"无 UI 访问文档 / access document without UI"。不适用：Document 内部功能导航。
  Trigger: UIDocument / Document / IExternalDBApplication。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.4（约p057-060）
tags: [revit-api, uidocument, document, architecture, revit-concepts]
related_skills: []
---

# Document 与 UIDocument 的职责分工

## R — 原文 (Reading)

> Autodesk.Revit.UI.UIDocument 为文件提供用户界面级的界面访问，如选择内容时提示用户作出选择及选取点。
> Autodesk.Revit.DB.Document 提供对文件级的所有其他属性的访问。
>
> — 宦国胜, 第1章 1.4（约p057-060）

---

## I — 方法论骨架 (Interpretation)

Revit API 的顶层是四个对象，两两配对成"应用层/文档层 × 数据层/UI 层"：

| | 数据层 (DB) | UI 层 |
|---|---|---|
| **应用级** | Application | UIApplication |
| **文档级** | Document | UIDocument |

分工规则一句话：**凡是"看得见、点得着"的操作走 UI 列，凡是"模型数据"的操作走 DB 列**：

- UIDocument：选择拾取（Selection、PickObject）、取点、激活视图等界面交互；外部命令里从 `commandData.Application.ActiveUIDocument` 拿到它。
- Document：图元检索、编辑、参数、保存等一切文件级数据操作。
- 两者之间可互换：`uiDocument.Document` 从 UI 文档下钻到数据文档。

这条边界决定了运行环境约束：在 **IExternalDBApplication**（无 UI 的数据库级应用）里没有 UIDocument 可用，只能通过 `Application.Documents` 枚举 DB 层文档——此时任何拾取/提示类 API 都不可调用。初学者常把两层混用导致编译失败（DB 层调 UI 方法）或运行异常（无 UI 语境调 UI 对象）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: Hello World 的入口
- **问题**: 第1章 1.2.2 的第一个命令要访问当前文档。
- **方法论的使用**: 经 `commandData.Application.ActiveUIDocument` 取得 UIDocument（外部命令天然有 UI 语境），再下钻 `.Document` 做数据操作。
- **结论**: 外部命令 = UIApplication → UIDocument → Document 的标准下钻链。
- **结果**: 首个示例即建立四对象的正确获取顺序。

### 案例 2: 无 UI 应用访问文档
- **问题**: IExternalDBApplication（无 UI 语境）里如何访问打开的文档？
- **方法论的使用**: 不能用 UIDocument，改经 `Application.Documents` 集合枚举 DB 层 Document。
- **结论**: 对象层级边界决定可用 API；无 UI 环境只有 DB 列。
- **结果**: 数据库级应用正常遍历文档工作。

### 案例 3: 全书命令的通用结构
- **问题**: 所有示例命令都同时需要用户选择和模型修改。
- **方法论的使用**: 统一模式——选择/拾取经 UIDocument，过滤/编辑经 Document。
- **结论**: "UI 操作走 UIDocument、模型操作走 Document"是全书通行的分工律。
- **结果**: 示例代码结构一致，职责清晰。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 刚接触 Revit API，分不清四个顶层对象该拿哪个、从哪拿。
2. 在 DB 层应用（IExternalDBApplication、批处理、云渲染回调）里调 UI API 失败。
3. 代码里对 Document 调选择/拾取方法——层级用错了。
4. 需要判断某个需求（弹提示？取选集？改参数？保存？）该挂在哪个对象上。

### 语言信号 (用户的话里出现这些就应激活)

- "UIDocument Document 区别 / difference between UIDocument and Document"
- "ActiveUIDocument 怎么拿 / get active ui document"
- "没有界面怎么操作文档 / access document without UI context"
- "Application UIApplication 区别 / application vs uiapplication"

### 与相邻 skill 的区分

- 与 `revit-document-function-map` 的区别: 本 skill 划定 Document 与 UIDocument 的层级边界；功能地图讲进入 Document 之后九域怎么导航。先分对象，再进地图。
- 与 `revit-selection-pickobject-filter` 的区别: 拾取 API 全部挂在 UIDocument 上是本分工律的具体体现；拾取模式的选择是下一层的决策。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定操作层级**：需求是界面交互（选择/拾取/提示/激活视图）还是数据操作（过滤/编辑/保存/参数）？
   - 完成标准: 每个操作标注 UI 或 DB。

2. **确认运行语境**：当前代码处于外部命令（有 UI 语境）还是 DB 级应用/后台（无 UI）？
   - 完成标准: 语境明确；无 UI 语境中所有 UI 类调用被识别为非法。
   - 判停条件: 若无 UI 语境且有交互需求 → 需求本身不成立，改设计（预设参数/配置文件）而非硬调 UI。

3. **按层取对象**：外部命令走 `ActiveUIDocument` → `.Document` 下钻；DB 应用走 `Application.Documents` 枚举。
   - 完成标准: 对象获取路径与语境匹配，UI 与 DB 调用各归其列。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已确定用 Document，讨论其内部能力分布——转功能地图 skill。
- 讨论具体拾取/选择 API 用法——转选择集 skill。

### 作者在书中警告的失败模式

- 无 UI 语境使用 UIDocument/拾取 API——直接失败。
- 在 Document 上找选择/拾取方法——层级错配，编译期即可暴露。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：四对象结构沿用至今；新版增加了更多 UI 层扩展点（如 DockablePane、UI 事件），但 UI/DB 分离的根本架构未变。

### 容易混淆的邻近方法论

- "UIDocument 包装 Document"容易让人以为 UI 文档是数据文档的升级版——它是**同一下层数据的两个视图**，UI 层不含任何模型数据，别把模型状态存在 UI 对象上。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
