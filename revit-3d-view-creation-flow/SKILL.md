---
name: revit-3d-view-creation-flow
description: |
  当需要通过 API 创建三维视图（透视/等轴测）并控制剖面框/裁剪框时调用。
  不适用于：二维视图创建、视图分类判别。
  关键 trigger 信号："创建三维视图 create 3d view"、"轴测图 isometric"、"透视图 perspective"、"剖面框 section box"、"裁剪框 crop box"、"ViewOrientation3D 设观察方向"。
  核心流程：筛选 ViewFamilyType → CreatePerspective/CreateIsometric → SetOrientation 设观察方向 → 控制 SectionBox/CropBox。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.6.2节三维视图 (约p120-125)
tags: [3d-view, creation, section-box, crop-box, view-orientation, revit-api]
related_skills:
  - slug: revit-view-classification-paths
    relation: composes-with
  - slug: revit-2d-view-creation-paths
    relation: contrasts-with
---

# 三维视图创建与剖面框控制流程

## R — 原文 (Reading)

> View3D.CreatePerspective 和 View3D.CreateIsometric 需要三维 ViewFamilyType……创建视图之后，可以调整裁剪框，以查看模型的不同部分……剖面框不同于裁剪框，它可以随模型旋转和移动。
>
> — 宦国胜, 第2章 2.6.2节三维视图 (约p120-125)

---

## I — 方法论骨架 (Interpretation)

三维视图创建与范围控制的主线：

1. **筛选 ViewFamilyType**：用 FilteredElementCollector 筛选三维视图族类型（ViewFamilyType，ViewFamily==ThreeD）。
2. **创建视图**：`View3D.CreatePerspective(doc, viewFamilyTypeId)` 创建透视图，`View3D.CreateIsometric(doc, viewFamilyTypeId)` 创建等轴测。
3. **设观察方向**：API 不支持直接修改视图坐标系，用 `ViewOrientation3D` 设置观察方向（eye/target/up 三个点）。
4. **范围控制**：
   - **裁剪框（CropBox）**：控制视图可见区域的边界框，不随模型旋转。
   - **剖面框（SectionBox）**：可随模型旋转和移动，赋 `BoundingBoxXYZ`（坐标经 Transform 转全局）得剖切轴测图；赋 null 隐藏。注意：仅当属性对话框中勾选 Section Box 时赋值才有效。

关键判别：透视视图 Scale 恒为 0（无比例概念），等轴测 Scale 是模型尺寸/视图尺寸比——据此可判断视图性质。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 生成剖切轴测图

- **问题**: 如何用 API 生成剖切轴测图？
- **方法论的使用**: CreateIsometric 建正等轴测后给 View3D.SectionBox 赋 BoundingBoxXYZ（坐标经 Transform 转全局）即得剖切轴测
- **结论**: 剖面框赋值实现剖切效果
- **结果**: 成功生成剖切轴测图

### 案例 2: 透视视图比例读出为 0

- **问题**: 透视视图为什么读出比例为 0？
- **方法论的使用**: 透视视图 Scale 恒为 0（无比例概念），等轴测 Scale 才是模型尺寸/视图尺寸比
- **结论**: 据 Scale 值可判断视图性质而无需查询类型
- **结果**: 通过 Scale 判别透视/等轴测

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要通过 API 自动生成轴测图或透视图
2. 需要生成剖切轴测图（带剖面框）
3. 需要控制三维视图的观察方向
4. 透视视图 Scale 读出为 0，需要诊断
5. 剖面框赋值后不生效，需要检查属性对话框是否勾选

### 语言信号 (用户的话里出现这些就应激活)

- "创建三维视图 / create 3d view / 轴测图 isometric"
- "透视图 / perspective view"
- "剖面框 / section box / 剖切轴测"
- "裁剪框 / crop box / 裁剪范围"
- "ViewOrientation3D / 观察方向"

### 与相邻 skill 的区分

- 与 `revit-view-classification-paths`：本 skill 依赖其分类路径来定位三维 ViewFamilyType，组合完成"选类型→建视图"；本 skill 负责创建与范围控制，不重复分类逻辑。
- 与 `revit-2d-view-creation-paths`：二者是对立分支——本 skill 管三维视图（CreatePerspective/CreateIsometric），后者管平面/剖面/立面/图纸等二维入口，按需求维度选择。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **筛选三维 ViewFamilyType**
   - `FilteredElementCollector(doc).OfClass(typeof(ViewFamilyType)).Where(x => x.ViewFamily == ViewFamily.ThreeD)`
   - 完成标准: 获取到三维视图族类型 Id

2. **创建视图并设观察方向**
   - `View3D view = View3D.CreateIsometric(doc, viewFamilyTypeId);`
   - `view.SetOrientation(new ViewOrientation3D(eye, forward, up));`
   - 完成标准: 视图创建且观察方向正确

3. **控制剖面框/裁剪框**
   - 剖切轴测：`view.SectionBox = boundingBoxXYZ;`（坐标经 Transform 转全局）
   - 隐藏剖面框：`view.SectionBox = null;`
   - 注意：剖面框赋值仅在属性对话框勾选 Section Box 时有效
   - 完成标准: 范围控制符合预期
   - 判停条件: 若剖面框赋值不生效，检查属性对话框是否勾选 Section Box

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 二维视图创建（平面/剖面/立面）——走对应的 Create 方法
- 视图分类判别——走 revit-view-classification-paths

### 作者在书中警告的失败模式

- 剖面框赋值在属性对话框未勾选 Section Box 时不生效
- API 不支持直接修改视图坐标系——只能用 ViewOrientation3D 设观察方向
- 透视视图 Scale 恒为 0——不要误判为错误

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增三维视图 API（如导航点、相机焦距控制）

### 容易混淆的邻近方法论

- 剖面框（SectionBox，可随模型旋转移动）vs 裁剪框（CropBox，固定边界）——两者行为不同
- CreatePerspective（透视图）vs CreateIsometric（等轴测）——前者有消失点，后者无

---

## 相关 skills

- **revit-view-classification-paths**（composes-with）：创建前需按分类路径筛选三维 ViewFamilyType，二者组合成"分类→创建"流程。
- **revit-2d-view-creation-paths**（contrasts-with）：三维视图与二维视图创建是并列分支，按视图维度择一而行。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
