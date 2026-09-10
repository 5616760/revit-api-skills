---
name: revit-geometry-options-visibility
description: |
  几何提取缺东西（缺中心平面/隔热层、梁楼梯只有粗轮廓）时查 Options 四属性：ComputeReferences 默认 false（要创建参照才开）；IncludeNonVisibleObjects 默认 false（条件几何被排除，设 true 返回）；View 指定后覆盖 DetailLevel；DetailLevel 默认 Medium，需 Fine。不适用于：取不到几何本身、需原始几何（GetOriginalGeometry）。
  Trigger："提取缺几何"、"geometry missing center plane"、"详细程度不够"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.3（约 p216–p217）
tags: [geometry-options, detaillevel, visibility, includenonvisibleobjects, troubleshooting]
related_skills: []
---

# 几何提取必须考虑可见性与详细程度（Options 四属性语义）

## R — 原文 (Reading)

> 选项（Options）。ComputeReferences：指示是否求出几何参照。默认值为 false。IncludeNonVisibleObjects：表示还包括默认视图中不可见的几何对象。View：从指定视图获取几何信息。DetailLevel：表示建议的详细程度。默认为"Medium"。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.3 节选项与详细程度（约 p216–p217）

---

## I — 方法论骨架 (Interpretation)

同样的"取几何"，返回内容由一份 Options 参数决定，四个属性四个坑：

ComputeReferences——要不要在几何上求参照（Reference）。默认 false；只有当你需要拿这些面/边去创建其他图元时才开 true，开了会显著变慢，别默认开。

IncludeNonVisibleObjects——要不要"默认视图中不可见"的几何。默认 false，于是族实例中心平面、MEP 风管隔热层这类"条件几何"会被排除——最常见的"几何缺失"就是这么来的，改 true 就回来了。

View——指定视图。一旦指定，提取结果按该视图的显示设置来，并且会覆盖 DetailLevel 的设置。

DetailLevel——详细程度。默认 Medium，梁/楼梯会返回粗略轮廓，要精确几何得设 Fine。

这套语义矩阵是"几何缺什么"类故障的排查清单：先看是不是被可见性过滤了，再看详细程度够不够，再想是否误开了重开关。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 柱中钢筋长度计算（3.7.2 代码 3-56）
- **问题**: 计算柱内钢筋长度需要精确几何。
- **方法论的使用**: Geometry.Options 设 ComputeReferences=true、DetailLevel=Fine。
- **结论**: 精确计算场景必须显式开 Fine。
- **结果**: 见 s3-c01。

### 案例 2: 概念设计 Form 表面分割（3.4）
- **问题**: 遍历概念体量的表面做分割。
- **方法论的使用**: GeometryOptions 设 ComputeReferences=true。
- **结论**: 需要创建参照的遍历要开 ComputeReferences。
- **结果**: 见 s2b-f20。

### 案例 3: MEP 族连接件创建（第4章）
- **问题**: 在族连接件面上创建连接。
- **方法论的使用**: Options.ComputeReferences=true + View=ActiveView 组合。
- **结论**: View 与 ComputeReferences 常搭配使用。
- **结果**: 见 s4-c04。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 提取结果缺几何：中心平面、隔热层、次要构件不出现。
2. 梁/楼梯的几何太"粗"，只有大致轮廓。
3. 不确定要不要开 ComputeReferences，担心性能。
4. 提取结果的详细程度与视图有关，换视图结果变了。

### 语言信号 (用户的话里出现这些就应激活)

- "提取结果里缺了中心平面/隔热层" / "geometry extraction is missing the center plane / insulation"
- "为什么梁的几何这么粗" / "why is the beam geometry so coarse"
- "要不要开 ComputeReferences" / "should I set ComputeReferences true"
- "IncludeNonVisibleObjects 是什么" / "what does IncludeNonVisibleObjects do"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **核对 Options 四属性**
   - 完成标准: 逐项确认 ComputeReferences（仅创建参照时 true）、IncludeNonVisibleObjects（缺条件几何时 true）、View（是否覆盖 DetailLevel）、DetailLevel（精度需求对应 Fine/Medium/Coarse）。

2. **按症状调参**
   - 完成标准: "缺几何"→ 先开 IncludeNonVisibleObjects；"太粗"→ 设 DetailLevel=Fine；"要创建参照"→ 开 ComputeReferences；三者不冲突可同时设置。
   - 判停条件: 若调参后仍缺，跳到步骤 3 排查是否属于"原始几何"需求。

3. **区分原始几何需求**
   - 完成标准: 若缺的是"连接/切割前的几何"，确认改用 FamilyInstance.GetOriginalGeometry()，而非继续调 Options。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 取到的几何"完全不对"（类型、位置错误）：那是遍历流程或坐标系问题，不是 Options 语义问题。
- 需求是原始几何：改入口 GetOriginalGeometry，Options 只是辅助。

### 作者在书中警告的失败模式

- 默认 DetailLevel=Medium 让梁/楼梯只返回粗略几何，算量结果偏差却无从察觉。
- 误开 ComputeReferences 拖慢大规模遍历——它是重开关，按需开。
- View 一旦指定会覆盖 DetailLevel，两者同时设置时以 View 为准，容易产生"我设了 Fine 怎么还是粗"的困惑。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：DetailLevel 的 Medium 默认值在新版本中延续；但新版本对条件几何（如中心平面）的过滤规则可能与视图状态联动更细，极端场景应以实测为准。

### 容易混淆的邻近方法论

- IncludeNonVisibleObjects 与视图可见性：前者让"默认视图不可见"的几何出现在结果里，后者是 UI 层的显示开关，别混用。
- Options.View 与 View.GetTemplateSetting：指定视图是"按该视图规则提取"，不是"只提取该视图能选的图元"。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
