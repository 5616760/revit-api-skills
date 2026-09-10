---
name: revit-taskdialog-component-rules
description: |
  当写 TaskDialog、纠结标题/按钮怎么写规范时调用。标题格式'<功能> — <短标题>'；MainInstructions 一句话为默认操作；按钮语义：OK 仅当问题可用 OK 回答、Canc
  el 仅当真可取消、Close 用于纯信息框；DNSMA 标准措辞 'Do not show me this message again'（不缩写）。不适用：多控件输入表单（用自定义 Dialog）。
  trigger：TaskDialog、标题格式、确定取消关闭按钮、task dialog button rules。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录 E.8.7 任务对话框（约p442-447）
tags: [task-dialog, ui-guidelines, modal-dialog, button-semantics]
related_skills: []
---

# 任务对话框组件使用规则

## R — 原文 (Reading)

> 每个主要按钮类型的正确用法说明：确定按钮——仅用于任务对话框所提出的问题可以用"OK"来回的情况下。取消按钮——仅用于一个任务真正可以被取消。关闭按钮——用于任何纯信息任务对话框。
>
> — 宦国胜, 附录 E.8.7 任务对话框（约 p442–p447）

---

## I — 方法论骨架 (Interpretation)

TaskDialog 是 Revit 风格的统一模态消息框，组件有固定 12 元素（Title、MainInstructions、MainContent、ExpandedContent、VerificationText(DNSMA)、CommandLinks、CommitButtons 等）。规范的核心是**每个元素都有语义要求**：

1. **标题格式**：`"<功能名称> — <短标题>"`，首字母大写。不能只用动词（如"导出"），也不要让标题重复命令链接的文本。例：`"DWG 导出 — 选择版本"`。
2. **主指令（MainInstructions）**：一句话说明用户要做什么；"第一行"即默认操作。
3. **按钮语义**（最容易错）：
   - **OK（确定）**：仅当问题确实能用"OK"回答（如"处理完成"）。
   - **Cancel（取消）**：仅当任务真的可以被取消（有可中止的进行中操作）。
   - **Close（关闭）**：用于纯信息对话框（没有操作、只是通知）。
   - **Yes/No/Retry**：按问句语义配对使用。
4. **DNSMA 复选框**：标准措辞是 `Do not show me this message again`——**不缩写**成 Don't。

判断主按钮的提问：**"用户对这个框能做什么？"** 只读通知→Close；可中止→Cancel；确认继续→OK/Yes。选错按钮类型是新手最常见的 UI 违规。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 按钮语义规范（附录 E.8.7）
- **问题**: OK、Cancel、Close 三个按钮什么时候用哪个？
- **方法论的使用**: 按"能否用 OK 回答/能否真取消/是否纯信息"三种提问场景分配。
- **结论**: 按钮类型由对话语义决定，不是随手放。
- **结果**: 对话框与用户预期一致，符合 Autodesk 指南。

### 案例 2: Revit 风格 TaskDialog 的两种创建方式（s1-c12）
- **问题**: 代码里怎么创建一个符合规范的任务对话框？
- **方法论的使用**: 用 `TaskDialog` 类按 12 元素填充（Title/MainInstruction/MainContent/命令链接等）。
- **结论**: 组件是标准化的，代码结构固定。
- **结果**: 插件对话框外观与 Revit 原生一致。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写完插件要点弹提示/确认，想知道 TaskDialog 怎么写才规范。
2. 纠结标题写"导出 DWG"还是"DWG 导出 — 选择版本"。
3. 不知道弹确认该用 OK 还是 Yes，或警告该用什么按钮。
4. 想让用户勾选"不再提示"（DNSMA），不知道标准措辞。

### 语言信号 (用户的话里出现这些就应激活)

- "TaskDialog 怎么写 / 标题怎么写"
- "确定还是取消还是关闭按钮"
- "Do not show me this message again"
- "任务对话框组件"
- "task dialog / main instructions / commit buttons / DNSMA checkbox / button type"

### 与相邻 skill 的区分

- 与 `revit-taskdialog-vs-dialog` 的区别: 本 skill 是"用 TaskDialog 时各组件怎么写"，vs-dialog 是"该用 TaskDialog 还是自定义 Dialog"——先决策再写规则。
- 与 `revit-ribbon-control-types` 的区别: Ribbon 是命令入口，TaskDialog 是消息反馈，两者都是 UI 规范但作用域不同。
- 与 `revit-execute-parameter-semantics` 的区别: 命令失败时用 message+elements 触发系统错误框，那与手写 TaskDialog 是两条反馈路径（前者自动，后者显式）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定对话框的对话语义**
   - 完成标准: 写清"这个框想从用户那里得到什么"：纯通知 / 可取消确认 / 继续/中止选择 / 选择选项（命令链接）。据此确定主导按钮（Close / Cancel / OK/Yes/No / CommandLinks）。

2. **按 12 元素规则填充**
   - 完成标准: Title 用"<功能> — <短标题>"格式；MainInstructions 一句话；按钮按语义选；DNSMA 用标准措辞 "Do not show me this message again"（不缩写）。给出 TaskDialog 代码（Show / ShowDialog）。
   - 判停条件: 若需要文本框/下拉/多复选框等输入控件 → 停，指路 `revit-taskdialog-vs-dialog` 改用自定义 Dialog。

3. **复核并验证显示**
   - 完成标准: 逐条对照规范清单（标题格式、按钮语义、DNSMA 措辞、主指令位置）通过；运行弹出确认外观与行为。给用户一份组件-规范对照表。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 多控件输入表单（文本框/下拉/复杂布局）——TaskDialog 装不下。
- 非模态窗口、进度显示（带取消按钮的进度对话框）——那是 External Events/非模态框架。

### 作者在书中警告的失败模式

- 标题只用动词或重复命令文本 → 不符合"功能 — 短标题"格式。
- 给纯信息框放 OK（应放 Close）→ 用户以为有操作要确认。
- 给不可取消的流程放 Cancel 按钮 → 点了也没用，误导用户。
- DNSMA 写成 "Don't show me..." → 与官方标准措辞不一致。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 指南：新版 TaskDialog 支持更多样式（如 `SetFooterText`、图标控制），但核心 12 元素规则不变。
- 指南以英文界面为准，中文本地化标题的格式（破折号、字符数）需译者折算，书未给出中文规范。

### 容易混淆的邻近方法论

- `TaskDialog`（Revit API，固定 12 元素）vs `MessageBox`（WinForms，极简）——用 MessageBox 会丢失 Revit 外观与 DNSMA 能力。
- "命令链接（CommandLinks）"vs "按钮（CommitButtons）"：命令链接是整行可点击说明（多个操作并列时用），提交按钮是底部标准按钮——别混用。

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
