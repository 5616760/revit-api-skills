---
name: revit-grid-curve-creation
description: |
  创建或读取轴网（Grid）时。读取分支：Grid.IsCurved=true 时 Curve 是 Arc，false 时是 Line——先判别再转型读取半径/长度。创建约束：用于创建轴网的弧线或直线必须在水平面内，竖直面曲线直接被拒绝。批量创建：收集 CurveArray 后一次 NewGrids(curveArray)，轴网名称自动按数字/字母序生成，事后可改 grid.Name。
  不适用于：读取洞口边界（走 revit-opening-boundary-reading）、标高/轴线处理。
  Trigger："怎么创建轴网"、"读弧形轴网的半径"、"NewGrid 失败"、"批建轴网"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.5.2 轴网（p197–p198）
tags: [grid, axis-grid, iscurved, arc-line, creation-constraints]
related_skills:
  - slug: revit-element-geometry-extraction
    relation: contrasts-with
---

# 轴网 Grid 曲线类型判断与创建流程

## R — 原文 (Reading)

> Grid 类 Curve 属性获取一个对象，该对象代表轴线几何形状。如果 IsCurved 返回 true，则 Curve 属性为 Arc 类对象；如果 IsCurved 返回 false，则 Curve 属性为 Line 类对象。用于创建轴网的弧线或直线必须在水平面内。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.5.2 节轴网（p197–p198）

---

## I — 方法论骨架 (Interpretation)

轴网（Grid）的几何是一句运行时判别：Grid.IsCurved 决定 Curve 属性的实际类型——true 时是 Arc（圆弧轴），false 时是 Line（直线轴）。读取代码必须两分支：转 Arc 读 Radius/Center，或转 Line 读 Length。先查布尔再转型，否则直接 cast 会崩。

创建侧有一条几何前置约束：创建轴网用的弧线或直线必须"在水平面内"——竖直面内画一条线去 NewGrid 会直接被拒绝，这是轴网特有的几何约束，不是通用的曲线创建规则。

批量场景有专门的 API：把所有轴线收集进 CurveArray，一次 NewGrids(curveArray) 批量创建，名称自动按数字/字母序生成，事后可改 grid.Name。判别属性的模式（IsCurved / IsRectBoundary / IsFoundationSlab）在同一对象模型中反复出现。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 读取弧形轴网（3.5.2）
- **问题**: 读取弧形轴网的半径与圆心。
- **方法论的使用**: 先查 IsCurved=true，再转 Arc 读 Radius/Center。
- **结论**: 读取必须两分支，不能盲目 cast。
- **结果**: false 时转 Line 读 Length。

### 案例 2: 创建轴网（3.5.2）
- **问题**: 用一条直线创建轴网。
- **方法论的使用**: 确认曲线在水平面内，NewGrid 创建。
- **结论**: 水平面是轴网创建的硬前置。
- **结果**: 竖直面曲线被拒绝。

### 案例 3: 批量建轴网（3.5.2）
- **问题**: 一次创建多根轴线。
- **方法论的使用**: 收集 CurveArray 调 NewGrids(curveArray)。
- **结论**: 批量 API 提升创建效率。
- **结果**: 名称自动编号，可改 grid.Name（呼应 revit-newfamilyinstances-batch-create 批量模式）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 从零创建轴网（直线轴、弧形轴、批量）。
2. 读取已有轴网的曲线类型、半径、长度。
3. NewGrid 失败（尤其怀疑曲线方向/平面问题）。
4. 批量生成轴网并控制命名。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么创建轴网" / "create a grid"
- "读弧形轴网的半径" / "get the radius of a curved grid"
- "NewGrid 为什么失败" / "why does NewGrid fail"
- "批量建轴网" / "create grids in batch"

### 与相邻 skill 的区分

- 与 `revit-element-geometry-extraction`：轴网几何由 Grid.Curve 直接给出，本 skill 不需遍历几何树；后者是通用提取流程——直接属性 vs 遍历，手段对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **创建侧：准备水平曲线**
   - 完成标准: 构造弧/直线并确认在水平面内（z 恒定）；竖直面曲线先处理，否则必然失败。

2. **创建侧：单根或批量**
   - 完成标准: 单根用 NewGrid；多根收集 CurveArray 用 NewGrids；名称按需在创建后改 grid.Name。

3. **读取侧：判别后转型**
   - 完成标准: 读 IsCurved；true 转 Arc 读 Radius/Center，false 转 Line 读 Length；验证转型成功、无 cast 异常。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 读取洞口边界：那是 revit-opening-boundary-reading。
- 处理标高/标头文字位置等轴网标注细节：不在本 skill 范围。

### 作者在书中警告的失败模式

- 用竖直面内曲线创建轴网：直接被拒绝——水平面是硬约束。
- 盲目把 Curve 当 Line 或 Arc 用：运行时类型由 IsCurved 决定，cast 前必须判别。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：IsCurved 语义与"水平面内"约束在后续版本保持；批量 NewGrids 的命名规则在新版本中仍自动生成，但可配置性有增强。

### 容易混淆的邻近方法论

- Grid.Curve vs 几何树遍历：轴网几何直接挂 Curve 属性，别把它当普通图元去遍历 Solid。
- 直线轴与弧形轴的读取：同一属性两个运行时类型，靠 IsCurved 二分支。

---

## 相关 skills

- **revit-element-geometry-extraction**（contrasts-with）：轴网几何由 Grid.Curve 直接给出，本 skill 不需遍历几何树；后者是通用提取流程——直接属性 vs 遍历，手段对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
