---
name: revit-command-visibility-mode
description: |
  当需要按文档类型或专业控制外部命令可见性时调用 .addin 声明式配置。
  不适用于：基于选中图元动态控制（用 IExternalCommandAvailability）。
  关键 trigger 信号："命令仅在族编辑器可用 command only in family editor"、"VisibilityMode"、"Discipline"、"静态可见性 static visibility"。
  核心：VisibilityMode（项目/族/无文档三态）与 Discipline（Architecture/Structure/MEP）两套正交枚举，.addin 声明式配置。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.3.4 (约p040)
tags: [visibility-mode, discipline, addin, command-availability, revit-api]
related_skills:
  - slug: revit-command-availability
    relation: contrasts-with
---

# VisibilityMode 与 Discipline 的场景可见性控制

## R — 原文 (Reading)

> VisibilityMode 成员：AlwaysVisible、NotVisibleInProject、NotVisibleInFamily、NotVisibleWhenNoActiveDocument。Discipline 成员：Any、Architecture、Structure、MEP。
>
> — 宦国胜, 第1章 1.3.4 (约p040)

---

## I — 方法论骨架 (Interpretation)

Revit 有两套正交的静态命令可见性控制枚举，通过 .addin XML 属性声明式配置：

**VisibilityMode（文档类型三态）**：
- `AlwaysVisible`：所有模式下命令都可用
- `NotVisibleInProject`：有活动项目文件时不可见（仅在族编辑器/无文档时可见）
- `NotVisibleInFamily`：有活动族文件时不可见（仅在项目文档时可见）
- `NotVisibleWhenNoActiveDocument`：无活动文档时不可见

**Discipline（专业规程）**：
- `Any`：所有规程中可用
- `Architecture` / `Structure` / `StructuralAnalysis` / `MEP`：限定专业

两套枚举正交组合，实现"仅在族编辑器中可用""仅在 MEP 项目中可用"等粗粒度控制。这是静态配置，适合按文档类型/专业做固定控制。若需基于选中图元等动态条件控制，则用 IExternalCommandAvailability 接口。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 命令仅在族编辑器中可用

- **问题**: 如何让一个命令仅在族编辑器中可用？
- **方法论的使用**: 在 .addin 中设置 VisibilityMode 为 NotVisibleInProject（项目文档时不可见）——比 IExternalCommandAvailability 更静态，适合按文档类型做粗粒度控制
- **结论**: NotVisibleInProject 使命令仅在族编辑器/无文档时可见
- **结果**: 命令在项目文档中不可见，仅在族编辑器中出现

### 案例 2: MEP 专用命令

- **问题**: 命令只在 MEP 专业项目中有意义
- **方法论的使用**: 在 .addin 中设置 Discipline 为 MEP
- **结论**: Discipline=MEP 限制命令仅在 MEP 规程项目可用
- **结果**: 命令在 Architecture/Structure 项目中不可见

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开发仅适用于族编辑器的命令
2. 开发仅适用于特定专业（MEP/Structure）的命令
3. 需要在 .addin 中配置命令可见性
4. 需要区分静态可见性（VisibilityMode/Discipline）vs 动态可用性（IExternalCommandAvailability）

### 语言信号 (用户的话里出现这些就应激活)

- "命令仅在族编辑器可用 / command only in family editor"
- "VisibilityMode / 可见性模式"
- "Discipline / 专业规程"
- "项目/族/无文档三态 / project family no-document"
- "静态可见性 / static visibility"

### 与相邻 skill 的区分

- 与 `revit-command-availability` 的区分：本 skill 用 .addin 声明式配置 VisibilityMode/Discipline，做静态粗粒度的可见性控制；后者用 IExternalCommandAvailability 接口在运行时基于条件（如选中图元）动态裁决命令可用性。前者无需代码逻辑、编译期即生效，后者需实现接口、每次按上下文求值，二者是静态声明与动态接口的对立关系。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定命令适用的文档类型和专业**
   - 文档类型：项目 / 族 / 无文档 / 全部
   - 专业：Any / Architecture / Structure / StructuralAnalysis / MEP
   - 完成标准: 明确目标文档类型和专业范围

2. **在 .addin 文件中配置 VisibilityMode 和 Discipline**
   - `<VisibilityMode>NotVisibleInProject</VisibilityMode>`
   - `<Discipline>MEP</Discipline>`
   - 完成标准: .addin 配置正确

3. **验证命令在目标场景可见、非目标场景不可见**
   - 完成标准: 命令可见性符合预期

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要基于选中图元动态控制可用性——用 IExternalCommandAvailability 接口
- 命令在所有场景都可用——默认 AlwaysVisible + Any 即可

### 作者在书中警告的失败模式

- 误用 VisibilityMode 为 NotVisibleInFamily 但命令需要在项目中可用 → 命令不可见
- 静态配置无法响应运行时条件变化

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增 VisibilityMode/Discipline 枚举值

### 容易混淆的邻近方法论

- VisibilityMode（文档类型三态）vs Discipline（专业规程）——前者按文档类型，后者按专业
- 静态 VisibilityMode/Discipline vs 动态 IExternalCommandAvailability——前者声明式，后者编程式

---

## 相关 skills

- **revit-command-availability**（contrasts-with）：本 skill 是 .addin 静态声明式可见性配置，后者是 IExternalCommandAvailability 运行时动态接口——粗粒度静态控制与细粒度动态裁决互为替代。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
