---
name: revit-model-vs-reference-line
description: |
  用户发现创建形状后原轮廓线消失了、想让形状可参数化调整（拖动驱动线形状跟随）、或问"API 有没有 ReferenceLine 类"时调用。不适用于：详图线/注释语境（那是注释曲线体系）、项目文档普通线、固定几何无需调整的场景。关键 trigger："轮廓线创建形状后不见了"、"被形状消耗"、"参照线"、"ChangeToReferenceLine"、"ReferenceLine 类"、"形状跟着线变"。核心：模型线会被形状"消耗"，参照线保留且驱动形状；API 无 ReferenceLine 类，需用 ModelCurve.ChangeToReferenceLine() 转换。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.4.1 点和曲线对象（p183）
tags: [modelcurve, referenceline, changetoreferenceline, conceptual-design, parametric, revit-api]
related_skills: []
---

# 模型线与参照线的区别及 ChangeToReferenceLine 转换

## R — 原文 (Reading)

> 形状可以使用模型线或参照线创建。模型线会在形状创建期间被形状"消耗"，不再作为独立图元存在。而参照线在形状创建后依然存在，移动它们形状也会更改。尽管 API 未提供 ReferenceLine 类，但可以用 ModelCurve.ChangeToReferenceLine() 将模型线转换为参照线。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.4.1 点和曲线对象（p183）

---

## I — 方法论骨架 (Interpretation)

在概念设计/族环境中创建形状（拉伸/放样等）时，驱动轮廓可以用两种线，命运完全不同：

- **模型线（ModelCurve）**：创建形状时被形状**"消耗"（吞噬）**——不再作为独立图元存在。适合一次性、固定几何。
- **参照线（Reference Line）**：创建形状后**依然存在**，且移动它形状**跟随更新**——参数化调整的关键。

**设计缺口与补救**：API 没有提供 `ReferenceLine` 类！但提供了 `ModelCurve.ChangeToReferenceLine()` 方法，把模型线**事后转换**为参照线。

推荐流程（参数化需求时）：
1. 用模型线绘制轮廓；
2. 调 `ChangeToReferenceLine()` 转成参照线；
3. 再创建形状——之后移动参照线，形状跟随更新。

决策一句话：**要参数化 → 参照线；只要固定几何 → 模型线**。这个"曲线类型选择决定后续行为"的决策在注释语境同样存在（详图曲线 vs 模型曲线）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 轮廓消失之谜
- **问题**: 创建形状后，原来的轮廓曲线不见了。
- **方法论的使用**: 识别出轮廓用的是模型线——被形状"消耗"。
- **结论**: 模型线在形状创建后不再独立存在，这是预期行为。
- **结果**: 解释了"消失"，需要保留时换参照线。

### 案例 2: 转参照线实现参数化
- **问题**: 希望形状后续可参数化调整。
- **方法论的使用**: 先画模型线 → ChangeToReferenceLine() 转参照线 → 创建形状。
- **结论**: 转换是"无 ReferenceLine 类"时代的标准补救路径。
- **结果**: 移动参照线，形状跟随更新。

### 案例 3: 注释语境的同类决策
- **问题**: 注释中画线，该用详图曲线还是模型曲线？
- **方法论的使用**: 曲线类型选择决定后续行为——与模型线/参照线的取舍同构。
- **结论**: "先选线类型、后定义行为"是跨语境的通用决策。
- **结果**: 在注释语境独立复现该决策模式。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 形状创建后轮廓线消失，需要解释与补救。
2. 要做一个参数化形状，拖动驱动线形状跟着变。
3. 找"ReferenceLine"类找不到，问怎么建参照线。
4. 在族/概念设计里画轮廓线，不确定用模型线还是参照线。

### 语言信号 (用户的话里出现这些就应激活)

- "轮廓线创建形状后不见了"（"the profile curve disappeared after creating the form"）
- "被形状消耗"（"consumed by the form"）
- "参照线"（"reference line"）
- "ChangeToReferenceLine"
- "想让形状可参数化调整"（"make the form parametric"）

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认参数化需求**
   - 形状以后要不要调整？要 → 参照线；不要 → 模型线即可。
   - 完成标准: 明确需求并选定线类型。

2. **绘制并转换**
   - 画 ModelCurve；要参数化 → 调 `ChangeToReferenceLine()`。
   - 注意：转换/创建在事务中进行，轮廓须闭合。
   - 完成标准: 轮廓线已按需求转为参照线或保持模型线。

3. **创建形状并验证**
   - 用轮廓创建形状；拖动参照线（或改驱动点）验证形状跟随更新。
   - 完成标准: 参数化联动生效，或固定几何符合预期。
   - 判停条件: 若用户需要的是点驱动参数化，转 `revit-referencepoint-curvebypoints`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 项目文档中创建普通模型线（非族/概念设计语境）。
- 详图/注释曲线——那是注释曲线体系，与模型线/参照线不同轨。
- 形状是固定几何、无需调整——用模型线更简单。

### 作者在书中警告的失败模式

- 以为参照线能凭空创建——API 没有 ReferenceLine 类，必须先建模型线再转换。
- 修改 ModelCurve 必须同步 Curve 与 SketchPlane（草图平面）——只改其一会导致不一致。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本仍无独立 ReferenceLine 类，ChangeToReferenceLine 延续；但部分转换边界（已参与形状的线）行为需实测。

### 容易混淆的邻近方法论

- 模型线（被形状消耗）与参照线（保留并驱动）是同一族 API 的两态，不是两类不同对象。
- 详图线是注释图元，与模型线完全不同的体系，别混。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
