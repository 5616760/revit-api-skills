---
name: revit-face-edge-loop-traversal
description: |
  需要把 Face 边界导出为闭合曲线环（DXF 外轮廓、压顶、洞口轮廓）时；面带洞（多个 EdgeLoop 区分外/内环）时；按固定参数方向拼接边界时。路径：Face.EdgeLoops → 每 EdgeLoop 遍历 Edge → Edge.AsCurve()（参数化 0–1），要与面同向用 AsCurveFollowingFace()。不适用于：只算面积（Face.Area）、取几何体本身（revit-element-geometry-extraction）。
  Trigger："导出面轮廓/边界曲线"、"extract face boundary loop"、"把洞口边变成多段线"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.2（约 p221–p223）
tags: [face-edges, edgeloops, boundary-extraction, curve-export, revit-geometry]
related_skills:
  - slug: revit-element-geometry-extraction
    relation: depends-on
---

# Face 边缘遍历与方向处理框架（EdgeLoops / Edge.AsCurve / AsCurveFollowingFace）

## R — 原文 (Reading)

> 边缘是给定表面的边界曲线。使用 EdgeLoops 属性迭代 Face 的所有边缘。每个循环都表示表面的一个封闭边界。边缘总是参数化为 0～1。使用 Edge.AsCurve() 和 Edge.AsCurveFollowingFace() 函数，可以获取 Edge 的 Curve 表示。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.2 节（约 p221–p223）

---

## I — 方法论骨架 (Interpretation)

面（Face）的边界不是一条杂乱的线段列表，而是"环套环"的三层结构：Face 上挂着一组 EdgeLoop（封闭边界循环），每个 EdgeLoop 里按顺序排着 Edge，每个 Edge 又用 AsCurve() 展开成带参数化的 Curve。多循环的情形就是"带洞的面"——外环加若干内环，天然能区分。

方向是这套框架里的隐藏要点：每条 Edge 始终参数化为 0～1（方向固定），但相邻两个面共享同一条几何边时，各自看到的参数方向可能相反。因此要"顺着面的方向"取边时用 AsCurveFollowingFace()，而拼接闭合环时按 Edge 参数化 0→1 的顺序推进即可保证环的连续性。还有一个语义坑：剖切视图会在模型面上制造"人为边"，它不是模型级的真实边界。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 体量阴影轮廓提取（3.7.6）
- **问题**: 把体量的受影面轮廓画成模型曲线。
- **方法论的使用**: 遍历 ExtrusionAnalyzer 结果 Face 的 result.EdgeLoops → 每条 Edge 调 AsCurve() 提取轮廓曲线并绘制。
- **结论**: "面边界 → 曲线"的提取思想可直接复用到任何需要轮廓的场合。
- **结果**: 见 s3b-c01 的完整案例。

### 案例 2: 楼梯足迹边界（3.10）
- **问题**: 获取楼梯跑的水平边界用于出图/碰撞。
- **方法论的使用**: StairsRun.GetFootprintBoundary() 返回边界曲线集合。
- **结论**: 同类思想由 API 直接封装成方法，无需手动遍历 Edge。
- **结果**: 属于"面边界 → 曲线"的第三种形态（见 revit-stairs-object-model）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 把某个面的外轮廓导出为闭合多段线（DXF、生成压顶、轮廓放样）。
2. 判断一个面是否带洞（EdgeLoops 数量 > 1 即有内环）。
3. 需要按与面一致的方向逐边读取边界（如法向相关的轮廓方向）。
4. 计算面边界的总长度或逐段曲线类型。

### 语言信号 (用户的话里出现这些就应激活)

- "把这个面的轮廓导出来" / "export the face outline as polylines"
- "面的边界环怎么遍历" / "iterate the edge loops of a face"
- "怎么判断面有没有洞" / "does this face have holes"
- "Edge 的方向怎么和面保持一致" / "get edge curve following the face"

### 与相邻 skill 的区分

- 与 `revit-element-geometry-extraction`：依赖其从 Element 取到 Solid/Face，本 skill 再从 Face 往下遍历 Edge/Curve。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **拿到目标 Face**
   - 完成标准: 已通过几何遍历（revit-element-geometry-extraction）或 API 返回值获得 Face 对象，确认非 null。

2. **遍历 EdgeLoops 与 Edge**
   - 完成标准: 遍历 Face.EdgeLoops；记录外环/内环数量；对每个 EdgeLoop 遍历其中全部 Edge，总数与面的边界数一致。
   - 判停条件: 若需求是"只取外轮廓"，跳过内环（非首个 EdgeLoop），继续到步骤 3。

3. **取 Curve 并拼接闭合环**
   - 完成标准: 每条 Edge 用 AsCurve()（或需与面同向时用 AsCurveFollowingFace()）转 Curve，按 0→1 参数化顺序串成首尾相接的闭合环；输出曲线列表并验证首尾点重合。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只要面的面积、质心等体量属性：用 Face.Area / ComputeCentroid，无需碰边。
- 只需要整面轮廓一条、忽略内部边界：筛选只取外环即可，但别误以为所有面只有一个环。
- 楼梯等有现成边界 API 的对象：优先用封装方法。

### 作者在书中警告的失败模式

- 相邻 Face 共享同一条 Edge 但参数方向可能相反——按"面"取方向必须 AsCurveFollowingFace。
- 剖切产生的"人为边"属于视图派生的几何，不是模型级边界，导出时会被混入。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：EdgeLoop 本身是 IDisposable 的句柄型对象，在 .NET 环境中遍历后应及时释放；部分新版 API 对 Face.EdgeLoops 的返回类型有封装调整。

### 容易混淆的邻近方法论

- AsCurve() 与 AsCurveFollowingFace()：前者给固定参数方向的曲线，后者给与面同向的曲线——用途不同，别混用。
- 外环 vs 内环：EdgeLoops 的第一个通常为外环，但代码不应依赖顺序，应按环面积或方向判断。

---

## 相关 skills

- **revit-element-geometry-extraction**（depends-on）：依赖其从 Element 取到 Solid/Face，本 skill 再从 Face 往下遍历 Edge/Curve。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
