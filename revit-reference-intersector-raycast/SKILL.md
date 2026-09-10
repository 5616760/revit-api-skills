---
name: revit-reference-intersector-raycast
description: |
  需要光线投影（垂直测距、遮挡检查、光线路径追踪）时。三步：准备 View3D → 构造 ReferenceIntersector（设 FindReferenceTarget 与过滤）→ Find()/FindNearest(start, dir) → 解析 ReferenceWithContext。仅与三维几何相交，需三维视图；FindNearest 只返回最近。不适用于：二维检测、链接文件/非活动设计选项几何。
  Trigger："光线投影/找最近楼面/遮挡"、"raycast / find nearest element"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.7（约 p224–p227）
tags: [referenceintersector, raycast, hit-testing, view3d, proximity-analysis]
related_skills: []
---

# 使用 ReferenceIntersector 进行光线投影分析的三步框架

## R — 原文 (Reading)

> 参照求交器（ReferenceIntersector）类用 Revit 选择工具找到图元与几何体。该类用指定方向光线查找命中几何体，只与三维几何体相交，需三维视图。Find() 返回匹配条件的 ReferenceWithContext 集合。FindNearest() 只返回离光源最近的相交参照。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.7 节由光线投影找出几何体（约 p224–p227）

---

## I — 方法论骨架 (Interpretation)

ReferenceIntersector 把"视线/光线"变成一次可编程的查询：从某一点沿某个方向发射一条虚拟光线，返回这条光线与三维几何的所有交点，或只返回最近的一个。

框架是固定的三步：第一步准备——必须有一个三维视图，构造器里放视图并指定 FindReferenceTarget（要命中什么：图元、面、边、还是最近面），再决定过滤集（哪些元素参与）；第二步执行——Find(startPoint, rayDirection) 拿全部命中，FindNearest 只拿最近的；第三步解析——返回的是 ReferenceWithContext 集合，需要再 GetReference() 拿到可用作后续操作的参照。

两个可调旋钮：光线方向就是 ViewDirection 或任意自定义向量，因此"垂直向下量到最近楼面"和"斜向查遮挡"是同一条框架的不同参数；用剖面框（section box）限制三维视图，可以缩小搜索范围。边界同样明确：只与三维几何相交，链接文件几何、非活动设计选项的图元不参与。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 天窗到最近楼面的垂直距离（3.7.7）
- **问题**: 测量天窗到其正下方最近楼面的距离。
- **方法论的使用**: 以天窗某点为起点、垂直向下为方向，FindNearest 拿最近楼面交点。
- **结论**: 垂直测距是 FindNearest 的直接用例。
- **结果**: 框架在同一小节被反复实例化。

### 案例 2: 相邻图元查找 / 光线路径追踪（3.7.7）
- **问题**: 找视线方向上被哪些构件遮挡、追踪管线路径。
- **方法论的使用**: 自定义射线方向，Find() 拿全部命中点序列。
- **结论**: 方向向量任意，框架不限于垂直。
- **结果**: 与第2章门窗可访问区域干扰检查（s1-c16）互为印证。

### 案例 3: FirstElementId 作为前置检索（2.3）
- **问题**: 为构造求交器准备过滤输入。
- **方法论的使用**: FirstElementId() 被明确标注为"用于存在性检查或 ReferenceIntersector 输入"的前置检索方式。
- **结论**: 输入准备是三步框架的第一步，书单独给出推荐检索 API。
- **结果**: 见 revit-collector-result-retrieval。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 量测垂直距离：天窗到楼面、灯具到吊顶、构件到楼板。
2. 遮挡/可见性检查：从某点看过去是否被挡、管线穿墙检测。
3. 查找某个方向上的相邻图元（最近的是谁）。
4. 光线路径追踪（采光、视线分析）。

### 语言信号 (用户的话里出现这些就应激活)

- "发一道光线找命中的构件" / "shoot a ray and find what it hits"
- "测天窗到最近楼面的距离" / "distance from skylight to nearest floor"
- "检查这个方向有没有遮挡" / "check occlusion along this direction"
- "找最近的相交图元" / "find nearest intersecting element"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **准备三维视图与输入**
   - 完成标准: 得到 View3D；需要缩小范围时设置剖面框；决定 FindReferenceTarget（图元/面/最近面/边）；确定过滤集合。
   - 判停条件: 若只有二维视图，先创建/获取一个 View3D 再继续。

2. **构造并执行投影**
   - 完成标准: new ReferenceIntersector(view3D)（含过滤与 target）；调用 Find(start, dir) 或 FindNearest(start, dir)；确认方向向量单位化。

3. **解析结果**
   - 完成标准: 遍历 ReferenceWithContext，逐个 GetReference() 得到参照；按需求取最近点或全命中列表；验证结果与人工预期一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 二维相交/命中检测：本 API 只与三维几何相交。
- 需要链接文件（RVT 链接）中的几何，或非活动设计选项下的图元：不返回这类命中。
- 需要的是"几何遍历式"的完整检查而非单点投影。

### 作者在书中警告的失败模式

- 忘记创建三维视图直接构造会失败——"只与三维几何相交"是硬前置。
- Find() 返回的是 ReferenceWithContext 集合，直接当 Reference 用会类型错误，必须先 GetReference()。
- 未限定过滤集时，命中集合可能巨大，性能失控。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 / .NET 4.0：ReferenceIntersector 的构造函数与 FindReferenceTarget 枚举在后续版本基本稳定；剖面框用法随 View3D API 演进，新版应核对 GetSectionBox/SetSectionBox 的调用形态。

### 容易混淆的邻近方法论

- Find() 与 FindNearest()：前者全命中（按距离排序），后者只要最近；拿全命中当最近用会导致逻辑错误。
- 视图方向（ViewDirection）与投影方向：ViewDirection 是构造时默认基线，Find/FindNearest 显式传的方向向量才是实际发射方向，两者可能不同。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
