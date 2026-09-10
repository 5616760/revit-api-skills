---
name: revit-geometry-utility-classes
description: |
  需要几何实用操作不知选哪个工具类时按任务决策：取复合主体特定面（外墙外表面）→ HostObjectUtils.GetSideFaces(wall, ShellLayerType.Exterior) 返回 Reference；体操作 → SolidUtils；连接/拆解与顺序 → JoinGeometryUtils；空间几何与边界 → SpatialElementGeometryCalculator（复用单实例，有内部缓存）。这些工具不适用于族文件。
  Trigger："取墙外表面"、"计算房间几何/边界"、"管理连接顺序"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.8（约 p228–p230）
tags: [hostobjectutils, solidutils, joingeometryutils, spatial-geometry, tool-selection]
related_skills:
  - slug: revit-reference-stable-handle
    relation: composes-with
---

# 几何实用程序类的选择决策（HostObjectUtils / SolidUtils / JoinGeometryUtils / SpatialElementGeometryCalculator）

## R — 原文 (Reading)

> HostObjectUtils 类提供快捷方法，定位复合主体的面。SolidUtils 类提供体操作的方法。JoinGeometryUtils 类含连接和拆解图元的方法。SpatialElementGeometryCalculator 可计算空间图元几何并获取边界图元关系。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.8 节几何实用程序类（约 p228–p230）

---

## I — 方法论骨架 (Interpretation)

面对"几何操作"需求时，不必每次都手动遍历 Solid，先看有没有现成的专用工具类——一张四栏分工表：

HostObjectUtils：快捷定位复合主体（墙、楼板等）的特定面，如外墙外表面、内表面，直接返回 Reference 集合；要日照/节能分析的外表面，一条 GetSideFaces(wall, ShellLayerType.Exterior) 就够，绕开"遍历 Solid.Faces 再判断朝向"的笨办法。SolidUtils：对体本身做操作（合并、剪切等体级运算）。JoinGeometryUtils：管图元之间连接与拆解、维护连接顺序。SpatialElementGeometryCalculator：算房间/空间这类空间图元的几何，并返回空间与边界图元的对应关系——它和 GetBoundarySegments 是互补的两条路，前者还能带出材质。

框架的决策规则：按任务类型（取面 / 做体运算 / 管连接 / 算空间）对号入座，并记住两条硬约束——这些工具"不适用于族文件"；SpatialElementGeometryCalculator 有内部缓存，应复用单实例，文件更改后缓存会自动失效。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 镜像墙定位参照面（2.5）
- **问题**: 镜像墙时需要墙的外侧面作镜像参照。
- **方法论的使用**: HostObjectUtils.GetSideFaces(wall, ShellLayerType.Exterior) 获取外墙参考面。
- **结论**: 取特定面用工具类而非手工遍历。
- **结果**: 见 s2-c01，与本章框架跨章节互证。

### 案例 2: 房间几何与边界材质（4.1 主题）
- **问题**: 计算房间几何、边界图元与材质关系。
- **方法论的使用**: SpatialElementGeometryCalculator 计算空间几何并返回边界图元。
- **结论**: 与 GetBoundarySegments 路径互补，各有适用面。
- **结果**: 见 revit-room-creation-boundary。

### 案例 3: 连接顺序管理
- **问题**: 构件连接顺序需要程序化管理。
- **方法论的使用**: JoinGeometryUtils 连接/拆解并管理顺序。
- **结论**: 连接是 Revit 中可编程的图元间关系。
- **结果**: 连接顺序改变会连锁影响连接处的几何。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 取墙/楼板的外表面或内表面用于日照、节能、净空分析。
2. 计算房间/空间的几何并查边界图元与材质。
3. 程序化管理构件的连接与拆解、调整连接顺序。
4. 对实体做合并/剪切等体级操作。

### 语言信号 (用户的话里出现这些就应激活)

- "取墙的外表面做分析" / "get the exterior face of a wall"
- "算房间的几何和边界" / "calculate room geometry and boundary elements"
- "管理梁柱的连接顺序" / "manage join order between elements"
- "对实体做合并/剪切" / "merge or cut solids"

### 与相邻 skill 的区分

- 与 `revit-reference-stable-handle`：工具类处理后的几何参照可交给其稳定句柄做后续引用，组合使用。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **识别任务类型**
   - 完成标准: 把需求归类为"取面 / 体级运算 / 连接管理 / 空间几何计算"之一，确定对应工具类。

2. **选择并调用工具**
   - 完成标准: 取面用 HostObjectUtils.GetSideFaces（指定 ShellLayerType）；空间计算 new SpatialElementGeometryCalculator 并复用单实例；连接用 JoinGeometryUtils；体运算用 SolidUtils。
   - 判停条件: 若目标是族文件内的对象，判停——工具不适用于族文件，改用几何遍历。

3. **处理结果与缓存**
   - 完成标准: 验证返回的 Reference 或几何可用；若反复调用空间计算，确认复用同一实例；文件变更后确认缓存已失效。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标是族文件内的几何：这些工具"不适用于族文件"。
- 需要任意面的完整集合（不只是特定外/内表面）：仍走通用几何遍历。
- 需要光线投影/遮挡查询：那是 revit-reference-intersector-raycast。

### 作者在书中警告的失败模式

- SpatialElementGeometryCalculator 有内部缓存：不复用单实例会性能浪费，文件更改后缓存被清除需重新计算。
- 楼板通常不作为房间的边界图元；墙切割出的洞口不会包含在返回的面中——边界结果与直觉可能不一致。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：ShellLayerType 枚举、HostObjectUtils 的重载形态在后续版本基本稳定；新版中 GetSideFaces 新增了返回类型与过滤的变体，落地前核对当前文档。

### 容易混淆的邻近方法论

- HostObjectUtils（取面）与 SolidUtils（改体）：一个拿 Reference，一个做体运算，对象层级不同。
- JoinGeometryUtils 与文档级删除级联：连接管理的拆解不等于删除，删宿主不会自动删连接伙伴。

---

## 相关 skills

- **revit-reference-stable-handle**（composes-with）：工具类处理后的几何参照可交给其稳定句柄做后续引用，组合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
