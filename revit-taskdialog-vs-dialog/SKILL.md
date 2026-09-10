---
name: revit-taskdialog-vs-dialog
description: |
  当要弹窗、纠结 TaskDialog 还是自定义 Dialog 时调用。纯提示/确认/选择→TaskDialog（统一外观、12 固定元素、含命令链接与 DNSMA）；需文本框/下拉/复选框/复杂布局
  →自定义 WPF/WinForm Dialog（失去原生外观换任意控件）；非模态→External Events 框架。不适用：非模态窗口。trigger：弹窗用什么、TaskDialog 够用吗、要
  输入框、taskdialog vs dialog、input form。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录 E.8.7 任务对话框（约p441-447）
tags: [task-dialog, dialog, ui-architecture, winforms, wpf]
related_skills: []
---

# 任务对话框（TaskDialog）vs 对话框（Dialog）

## R — 原文 (Reading)

> 任务对话框是一种模态对话框。它们有共同的一组控件，这些控件按一个标准顺序排列以保证一致的外观和感觉。
>
> — 宦国胜, 附录 E.8.7 任务对话框（约 p441–p447）

---

## I — 方法论骨架 (Interpretation)

"要弹窗"在 Revit 插件里有两条路，选择取决于**你需要什么控件**：

- **TaskDialog**：Revit 自带的统一模态框。控件集合是**固定的子集**——标题、主指令、主内容、命令链接、标准按钮、DNSMA 复选框等 12 元素，按标准顺序排列，保证和 Revit 原生弹窗长得一模一样。适合**信息提示、确认、简单选择**。
- **自定义 Dialog（WPF/WinForm）**：你自己写的窗口，**任意控件**随便放——文本框、下拉列表、复选框、表格、复杂布局。代价是**失去 Revit 原生外观**，要自己做风格统一。

决策树：

1. 只是"提示/确认/二选一"→ TaskDialog（省事、原生、规范）。
2. 需要任何"输入"或"复杂布局"→ 自定义 Dialog。
3. 需要非模态（窗口开着还能操作 Revit）→ TaskDialog/普通 Dialog 都不行，要用 **External Events 框架**（那是另一专题）。

一句话：TaskDialog 是"标准化的提示框"，Dialog 是"自由的面包车"——先问自己要不要装输入控件。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 输入表单决策（附录 E.8.7）
- **问题**: 插件需要一个含文本框、下拉、复选框的表单，能用 TaskDialog 吗？
- **方法论的使用**: 检查 TaskDialog 控件集合——无输入控件，判定必须用自定义 Dialog。
- **结论**: 控件能力边界决定框架选型。
- **结果**: 表单需求走 WPF/WinForm，TaskDialog 让位。

### 案例 2: 纯信息提示（s1-c12）
- **问题**: 只是告诉用户"导出完成"，要最省事的规范方案。
- **方法论的使用**: 用 TaskDialog 的 Title/MainInstruction/MainContent 展示。
- **结论**: 纯提示场景 TaskDialog 是标准答案。
- **结果**: 弹窗与 Revit 外观一致，零样式代码。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写一个"设置面板/输入参数"的窗口，不知道用 TaskDialog 还是自己画。
2. 想弹个"确认删除？"的框，想知道最规范省事的做法。
3. 尝试把文本框塞进 TaskDialog 失败，需要理解为什么。
4. 想要非模态的悬浮窗，确认这不是 TaskDialog/Dialog 能解决的。

### 语言信号 (用户的话里出现这些就应激活)

- "弹窗用什么 / TaskDialog 够用吗"
- "要一个带输入框的窗口"
- "TaskDialog 和 WinForm 区别"
- "非模态窗口怎么做"
- "taskdialog vs dialog / input form / winform or taskdialog / modal dialog"

### 与相邻 skill 的区分

- 与 `revit-taskdialog-component-rules` 的区别: 本 skill 是"选哪条技术路线"，component-rules 是"选定 TaskDialog 后各元素怎么写"。
- 与批次8外部事件专题（非模态对话框）的区别: 非模态需要 External Event 桥接 Revit 主线程，本 skill 只覆盖模态场景。
- 与 `revit-ribbon-panel-layout` 的区别: Ribbon 是命令入口面板，Dialog 是任务执行中的交互窗口，UI 层次不同。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **盘点交互需求**
   - 完成标准: 列出窗口需要的全部元素（文本提示/按钮/命令链接/输入控件）。判断是否有输入控件或复杂布局。无输入 → 走 TaskDialog；有输入 → 走自定义 Dialog。

2. **选择技术路线并实现**
   - 完成标准: TaskDialog 路线→用 TaskDialog/Show 或 CommandLinks 实现；自定义路线→给出 WPF/WinForm 窗口骨架，并提醒线程亲和：窗口打开期间不可跨线程访问 Revit 模型（模态 ShowDialog 才安全）。
   - 判停条件: 若需要"窗口开着还能继续操作 Revit" → 停，指路 External Events 外部事件框架（批次8）。

3. **验证与规范复核**
   - 完成标准: 模态框能弹出、交互正确；TaskDialog 路线复核 12 元素规范（见 component-rules）；自定义路线确认外观与 Revit 风格协调（主题色/字体）。给用户一句选型结论与理由。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 非模态窗口、后台进度框——External Events 范畴。
- 命令失败的系统反馈——用 message + Result（自动错误框），不是手写对话框。

### 作者在书中警告的失败模式

- 用 TaskDialog 硬塞输入功能 → 无此控件，白费功夫。
- 自定义 Dialog 非模态打开后直接操作文档 → 线程/事务异常（模型访问必须在 Revit 主线程，且非模态期间事务受限）。
- 对话框里没按 Revit 视觉规范做 → 与原生界面割裂，用户困惑。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版对外部事件桥接非模态 UI 的教程更完善（revit-external-events-nonmodal-dialog 提及但未展开），本 skill 仅指路。
- 书未讨论 WPF 数据绑定/国际化与 Revit 主题的适配，实操需补充。

### 容易混淆的邻近方法论

- `TaskDialog`（Revit 统一框）vs `MessageBox`（WinForms）vs `Dialog`（自定义）——三者能力递增、原生性递减，别只用 MessageBox 省事丢风格。
- "模态"（阻塞，不能操作 Revit）vs"非模态"（可并行操作）——这是选型的第一问题，先于 TaskDialog/Dialog。

---

## 相关 skills (阶段 3 填充)

- depends-on: {{}}
- contrasts-with: {{}}
- composes-with: {{}}

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
