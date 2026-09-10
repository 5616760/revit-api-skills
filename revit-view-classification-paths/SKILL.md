---
name: revit-view-classification-paths
description: |
  当需要分类/遍历/识别视图类型时调用三条并行分类路径。
  不适用于：创建视图（走 ViewFamilyType 路径）、视图内容操作。
  关键 trigger 信号："找出所有三维视图 find all 3d views"、"判断视图类型 classify view"、"ViewType Schedule ThreeD"、"ViewFamilyType"、"GetTypeId"。
  核心决策：粗筛用 ViewType 枚举；细分用 typeof 派生类；创建/复制用 GetTypeId→ViewFamilyType。三条路径答案不同，选错会漏判或误判。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.6.2节视图类型 (约p117-119)
tags: [view-classification, viewtype, viewfamilytype, view-traversal, revit-api]
related_skills: []
---

# 视图分类的三条决策路径

## R — 原文 (Reading)

> 在 API 中，有三种方法可用于分类视图。第一种方法是使用视图图元 View.ViewType 属性……第二种方法是按类的类型进行视图分类……第三种对视图进行分类的方法是使用 ViewFamilyType 类。
>
> — 宦国胜, 第2章 2.6.2节视图类型 (约p117-119)

---

## I — 方法论骨架 (Interpretation)

Revit 对同一视图对象有三套并行分类坐标，按精度递进选择：

1. **ViewType 枚举（粗筛）**：`view.ViewType == ViewType.ThreeD` 或 `ViewType.Schedule`。最粗粒度，适合按大类遍历（如"所有三维视图""所有明细表"）。
2. **派生类类型（细分）**：`view is View3D` / `view is ViewSchedule`。运行时类型判别，可区分透视图/等轴测等同大类下的子类型。
3. **ViewFamilyType（创建/复制用）**：`view.GetTypeTypeId()` → 获取 ViewFamilyType → 读 ViewFamily 枚举。这是最精细的分类，也是创建视图 API 唯一认的路径。

关键裁决：大多数视图创建方法需要 ViewFamilyType 的 Id，因此第三种路径在创建场景下最实用。选错路径会漏判或误判——如用 ViewType==ThreeD 无法区分透视图与等轴测。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 找出项目中所有三维视图

- **问题**: 如何找出项目中所有三维视图？
- **方法论的使用**: 粗筛用 ViewType==ThreeD → 需要区分透视图/等轴测时用 typeof(View3D) 运行时类型
- **结论**: 三条路径精度不同，选 ViewType 粗筛即可满足"找所有三维视图"
- **结果: 成功遍历所有三维视图

### 案例 2: 判断某视图是否为明细表

- **问题**: 判断某视图是否为明细表该用哪条路径？
- **方法论的使用**: 用 ViewType==Schedule 粗筛，或 typeof(ViewSchedule) 细分
- **结论**: ViewType 枚举足够判断明细表大类
- **结果**: 正确识别明细表视图

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要遍历项目中特定类型的视图（三维/平面/明细表等）
2. 需要区分透视图与等轴测等同大类下的子类型
3. 需要复制或创建视图，需要获取 ViewFamilyType
4. 视图遍历结果漏判或误判，需要诊断分类路径是否选错

### 语言信号 (用户的话里出现这些就应激活)

- "找出所有三维视图 / find all 3d views / 遍历视图"
- "判断视图类型 / classify view / view type"
- "ViewType / Schedule / ThreeD"
- "ViewFamilyType / GetTypeId / 创建视图"
- "透视图 等轴测 / perspective isometric"

### 与相邻 skill 的区分

- 本 skill 聚焦"视图分类的三条并行路径"（ViewType 枚举 / 派生类类型 / ViewFamilyType），是视图遍历与识别的独立方法论；它与其他 skill 无明显依赖、对比或组合关系（独立性强），不参与创建/编辑视图的链条。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定分类精度需求**
   - 粗筛（按大类）→ ViewType 枚举
   - 细分（同大类子类型）→ typeof 派生类
   - 创建/复制 → GetTypeId → ViewFamilyType
   - 完成标准: 明确精度需求

2. **按选定路径分类视图**
   - ViewType: `view.ViewType == ViewType.ThreeD`
   - 派生类: `view is View3D`
   - ViewFamilyType: `doc.GetElement(view.GetTypeTypeId()) as ViewFamilyType`
   - 完成标准: 正确分类视图

3. **创建场景下取 ViewFamilyType.Id 供 Create 方法使用**
   - 完成标准: 获取到 ViewFamilyType 的 Id

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 视图内容操作（如改视图范围、可见性图形替换）——走 View 对象属性
- 仅需判断"是否有视图"——直接查文档视图列表即可

### 作者在书中警告的失败模式

- 用 ViewType==ThreeD 无法区分透视图与等轴测 → 需要细分时漏判
- 创建视图时不用第三条路径（ViewFamilyType）→ 创建 API 不认

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增 ViewType 枚举值或 ViewFamily 类型

### 容易混淆的邻近方法论

- ViewType（枚举粗筛）vs ViewFamilyType（族类型精细分类）——前者按大类，后者按族类型
- ViewFamilyType（视图族类型）vs ViewFamily（视图族枚举）——前者是对象，后者是枚举

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
