---
name: revit-ribbon-control-types
description: |
  当在功能区加控件、纠结用按钮/下拉/分割/单选组、或遇 SplitButton 工具提示无效、StackedItems 报错时调用。选型：单动作→PushButton；并列→PulldownButto
  n；带默认动作→SplitButton（不设 ToolTip）；互斥→RadioButtonGroup（提示设在组内按钮）；输入→TextBox/ComboBox；堆叠→StackedItems（≤3
   个）；低频→Slide-out。不适用：命令逻辑。trigger：Ribbon 控件、分割按钮、单选按钮组、ribbon control types。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p044-053
tags: [ribbon, controls, ui-design, pushbutton, combobox]
related_skills: []
---

# Ribbon 控件类型与使用场景决策

## R — 原文 (Reading)

> 面板上可以包含按钮，大小均可，可以是简单的按钮，也可以是下拉式按钮或带有默认按钮的下拉分割按钮。除以上这些按钮外，面板还可以包含单选按钮组、组合框和文本框。面板中可以使用垂直分隔符将命令按逻辑分组。最后，面板还可以包括一个滑出式控件，通过单击面板底部来访问该控件。
>
> — 宦国胜, 第1章 1.3.8（约 p044–p053）

---

## I — 方法论骨架 (Interpretation)

Revit 的 Ribbon 面板是一个"控件仓库"，每种控件对应一种交互形态：

- **PushButton**：点击执行一个命令——最常用。
- **PulldownButton**：下拉列出一组同级命令（每次都是全新选择）。
- **SplitButton**：下拉分割按钮——**有默认项**，点主体直接执行默认项，点小三角展开其余项。注意：它的 ToolTip 设置无效。
- **RadioButtonGroup**：单选按钮组——互斥模式切换（如线框/着色/真实）。整个组不设工具提示，提示要设在组内每个 ToggleButton 上。
- **TextBox / ComboBox**：文本输入 / 下拉选择。
- **StackedItems**：把两个小控件并排堆叠（最多 3 个控件）。
- **Slide-out（滑出式）**：面板底部的隐藏区域，放低频命令/管理项。

两条选择主线：

1. **交互形态决定控件**：一个动作→PushButton；N 个并列→PulldownButton；有默认+更多→SplitButton；互斥→RadioButtonGroup；要输入→TextBox/ComboBox。
2. **特殊约束是红线**：SplitButton 别设 ToolTip、单选按钮组别整体设提示、RadioButtonGroup 不能放进 StackedItems、StackedItems 最多 3 个。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 面板创建用 PushButton（1.2.3）
- **问题**: 第一个面板放什么控件？
- **方法论的使用**: 用 `PushButtonData` 创建 PushButton，绑定命令类。
- **结论**: 单命令场景 PushButton 足够。
- **结果**: 面板随启动加载，按钮可点击（见 s1-c05）。

### 案例 2: 单选按钮组互斥模式（1.3.8）
- **问题**: 线框/着色/真实这类互斥模式用什么表达？
- **方法论的使用**: RadioButtonGroup + 组内 ToggleButton，提示设在按钮上而非组上。
- **结论**: 互斥语义由 RadioButtonGroup 承载。
- **结果**: 模式切换按钮行为符合预期，无多余 ToolTip 冲突。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 决定"这组命令用下拉还是分割按钮还是并列按钮"。
2. 要做一组互斥模式切换（如显示模式、视图样式）。
3. 给 SplitButton 设 ToolTip 不生效、StackedItems 放 4 个控件报错。
4. 想让两个小按钮并排省空间。

### 语言信号 (用户的话里出现这些就应激活)

- "功能区加按钮 / 下拉按钮 / 分割按钮 / 单选按钮组"
- "SplitButton 工具提示不显示"
- "两个按钮并排 / StackedItems 最多几个"
- "互斥的按钮组怎么做"
- "ribbon button / pulldown button / split button / radio group / stacked items / slide out panel"

### 与相邻 skill 的区分

- 与 `revit-ribbon-panel-layout` 的区别: 本 skill 是"用哪种控件"，panel-layout 是"控件在面板上怎么摆"（大按钮、三列、滑出面板位置）。
- 与 `revit-command-availability` 的区别: 本 skill 决定控件类型，availability 决定控件可点性（动态灰显）。
- 与 `revit-taskdialog-vs-dialog` 的区别: Ribbon 是命令入口区，TaskDialog 是消息反馈区——一个在前台调度、一个在结果呈现。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确交互形态需求**
   - 完成标准: 回答"这个控件是点一次、选一个、还是输入值"以及"是一对一还是一对多"。按形态映射候选控件类型，列出 1-2 个合理选项。
   - 判停条件: 若用户问的是“控件在面板上怎么摆、放哪”（大按钮/列数/滑出面板），停并指路 revit-ribbon-panel-layout。

2. **检查特殊约束**
   - 完成标准: 对照红线清单：SplitButton 不设 ToolTip；RadioButtonGroup 提示放在 ToggleButton；RadioButtonGroup 不进 StackedItems；StackedItems ≤3 控件。与用户确认约束不会破坏设计。

3. **生成创建代码并验证**
   - 完成标准: 给出对应 `RibbonXxxData` 创建代码（PushButtonData/PulldownButtonData/SplitButtonData/RadioButtonGroup 等），编译并运行验证显示与点击。给用户一句控件-场景速查结论。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 命令内部逻辑、事务、参数读写——控件只是入口，业务在别处。
- WPF/WinForm 自定义窗口内的控件——那是 Dialog 范畴。

### 作者在书中警告的失败模式

- SplitButton 设置了 ToolTip → 静默无效（API 忽略该设置）。
- RadioButtonGroup 整体设 ToolTip → 无效；正确做法是设在组内每个按钮。
- RadioButtonGroup 尝试堆叠 / StackedItems 超过 3 控件 → 运行时报错或布局异常。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版 Ribbon 增加更多样式（如按钮不同尺寸、`SetImage` 特性），控件行为基本兼容。
- 书未系统给"控件选型速查表"，只分散在 1.3.8 与附录——本 skill 的映射是对其的整理。

### 容易混淆的邻近方法论

- PulldownButton（无默认项，每次点开选）vs SplitButton（有默认项，点主体直执行）——区别在"是否存在默认动作"。
- RadioButtonGroup（互斥）vs 一组 PushButton（可同时激活）——别用按钮模拟互斥，没有互斥语义。

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
