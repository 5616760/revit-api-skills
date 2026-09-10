---
name: revit-sketch-sketchplane-model
description: |
  程序化绘制模型曲线/轮廓（任意平面，如坡屋顶面）或编辑草图轮廓时。工作模型：Plane(origin, normal) → SketchPlane.Create(doc, plane) → NewModelCurve(curve, sketchPlane)。Sketch 是瞬态图元，不可经 Document.Elements 枚举。ModelCurve 换平面须 SetPlaneAndCurve() 同步改 Curve 与 SketchPlane。不适用：详图线、楼板/屋顶轮廓（走各自 API）。
  Trigger："斜面上画曲线"、"曲线换平面"、"sketch plane"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.8（约 p231–p241）
tags: [sketch, sketchplane, modelcurve, plane, curve-creation]
related_skills:
  - slug: revit-face-edge-loop-traversal
    relation: contrasts-with
---

# 草图（Sketch）与草图平面（SketchPlane）的工作模型（含 SetPlaneAndCurve 一致性）

## R — 原文 (Reading)

> 要在 Revit 中创建图元或编辑它们的轮廓，需要首先创建草图对象。……Sketch 类表示用于创建三维模型的平面内的封闭曲线，其最重要的特性由 SketchPlane 和 CurveLoop 表示。Sketch 对象是瞬态图元，不能经 Document.Elements 枚举检索。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.8 节草图（约 p231–p233）

---

## I — 方法论骨架 (Interpretation)

Revit 里"画线"不是裸画一个 Curve，而是三层装配：先有一个 Plane（数学平面：Origin + Normal 定义局部坐标系），再用这个 Plane 造一个 SketchPlane（草图平面，它承载"线画在哪"的上下文），最后把曲线作为 ModelCurve 挂到 SketchPlane 上。要画在坡屋顶面上，就先把 Plane 的 Normal 指向屋顶法向，一切随之倾斜——这就是程序化在任意面绘制的原理。

这套模型有两条容易踩的约定：其一，Sketch 是瞬态图元，靠 Document.Elements 枚举永远找不到它，操作草图要按持有对象走；其二，SketchPlane 和 UI 里的"工作平面（Work Plane）"是两个概念，别混为一谈。

修改时还有一条一致性铁律：ModelCurve 想换平面，不能分开设 Curve 和 SketchPlane 两个属性——必须 SetPlaneAndCurve() 一次同时改，否则曲线与平面脱节，内部数据不一致，模型行为变得不可解释。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 族建模的轮廓定义（3.4）
- **问题**: 在族文件里创建拉伸/放样需要轮廓。
- **方法论的使用**: NewModelCurve + SketchPlane 定义轮廓曲线。
- **结论**: 草图平面是族建模轮廓的基础设施。
- **结果**: 见 revit-familyitemfactory-shape-creation、s2b-c08。

### 案例 2: 阴影轮廓绘制（3.7.6）
- **问题**: 把体量阴影轮廓画成模型曲线。
- **方法论的使用**: NewSketchPlane + NewModelCurveArray 生成模型曲线。
- **结论**: 分析结果可通过"平面→草图→曲线"落回模型。
- **结果**: 见 s3b-c01。

### 案例 3: 跨平面修改必须 SetPlaneAndCurve（3.8.3）
- **问题**: 修改 ModelCurve 跨到另一平面后模型行为异常。
- **方法论的使用**: 改用 SetPlaneAndCurve() 同时更新 Curve 与 SketchPlane。
- **结论**: 单独设置属性导致内部数据不一致。
- **结果**: 原 s3b-p02 并入本单元，作为一致性约束的完整教训。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在任意倾斜面（坡屋顶、斜面墙）上程序化绘制曲线。
2. 分析结果要落回模型成为可见的模型曲线（轮廓线、辅助线）。
3. 把已有的 ModelCurve 移到另一个草图平面。
4. 修改曲线或平面后出现"线的位置和预览不一致"等异常。

### 语言信号 (用户的话里出现这些就应激活)

- "在斜屋顶面上画一条线" / "draw a curve on the sloped roof"
- "创建草图平面再画线" / "create a sketch plane and draw model curves"
- "把这条线移到另一个平面" / "move this model curve to another plane"
- "SetPlaneAndCurve 什么时候用" / "when to use SetPlaneAndCurve"

### 与相邻 skill 的区分

- 与 `revit-face-edge-loop-traversal`：本 skill 把 Curve 画回模型（写），后者把面边界读出为 Curve（读），互为反向。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **定义平面**
   - 完成标准: 用 Plane.Create(origin, normal)（或 Face 派生）得到目标平面；确认 Normal 方向正确（决定线的"上方"）。

2. **创建 SketchPlane 并画曲线**
   - 完成标准: SketchPlane.Create(doc, geometryPlane) 成功；NewModelCurve(curve, sketchPlane) 返回有效 ModelCurve；曲线确实位于目标平面上。
   - 判停条件: 若需求是"把已有曲线换平面"，跳过画线，进入步骤 3 的修改流程。

3. **跨平面修改用 SetPlaneAndCurve**
   - 完成标准: 调用 SetPlaneAndCurve(newCurve, newSketchPlane) 一次性完成；验证曲线位置与平面一致；未单独设置 Curve 或 SketchPlane 属性。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 画详图曲线：注释详图线走 View + SketchPlane 的另一套创建 API（见 s3-p02）。
- 创建墙/楼板/屋顶的轮廓：系统族有自己的 Create 流程，轮廓不落成独立的 ModelCurve 集。
- 找文档里所有的 Sketch：枚举不到，这是设计使然，不是 bug。

### 作者在书中警告的失败模式

- 单独设置 ModelCurve.Curve 或 SketchPlane 属性会导致内部数据不一致，模型行为异常——必须 SetPlaneAndCurve()。
- Sketch 是瞬态图元，Document.Elements 枚举不到，靠枚举找草图必然空手而归。
- 把 UI 工作平面（Work Plane）当成 SketchPlane 混用，概念错位。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：SketchPlane.Create 的重载在新版本中仍是标准入口；SetPlaneAndCurve 的一致性契约保持稳定，但新版本对 ModelCurve 的编辑还提供了更多校验，落地方案应结合当前 API。

### 容易混淆的邻近方法论

- Plane 与 SketchPlane：Plane 是纯数学平面，SketchPlane 是模型里承载草图的图元，先 Plane 后 SketchPlane 不可逆序。
- 工作平面（Work Plane，UI 概念）与 SketchPlane（API 图元）：同名似义，接口层面完全不同。

---

## 相关 skills

- **revit-face-edge-loop-traversal**（contrasts-with）：本 skill 把 Curve 画回模型（写），后者把面边界读出为 Curve（读），互为反向。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
