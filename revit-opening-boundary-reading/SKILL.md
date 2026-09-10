---
name: revit-opening-boundary-reading
description: |
  读取洞口（Opening）几何边界时。先读判别属性 IsRectBoundary：true → Opening.BoundaryRect 返回 IList<XYZ> 顶点；false → BoundaryCurves 返回 CurveArray；选错读另一套得 null 而非异常。竖井洞口（Shaft Openings）无宿主，Host 恒为 null，遍历时要按类别跳过或特判。不适用于：创建洞口（走 NewOpening 重载 skill）、修改洞口轮廓。
  Trigger："BoundaryRect 返回 null"、"读洞口的边界"、"竖井洞口 Host 是空的"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.7 洞口（p154）
tags: [opening, boundary, isrectboundary, shaft, api-branching]
related_skills:
  - slug: revit-face-edge-loop-traversal
    relation: contrasts-with
  - slug: revit-new-opening-overloads
    relation: contrasts-with
---

# 洞口 Opening 几何边界读取分支

## R — 原文 (Reading)

> IsRectBoundary：识别洞口是否为矩形边界。true 意味着洞口有矩形边界，可从 Opening.BoundaryRect 属性获取 IList<XYZ> 集合；否则返回 null。false 时从 BoundaryCurves 属性获取 CurveArray 对象。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.7 节洞口（p154）

---

## I — 方法论骨架 (Interpretation)

洞口（Opening）的边界几何有两套互斥的读取属性，用哪个由判别属性 IsRectBoundary 决定：矩形边界（true）走 BoundaryRect，拿到 IList<XYZ> 顶点集合；非矩形边界（false）走 BoundaryCurves，拿到 CurveArray 曲线集。两套属性只各自在一个分支里有值——选错一套读另一套，得到的是 null，不是空集，也不会抛异常。所以读洞口边界的正确姿势是"先判别、后读取"，反过来就是空引用事故。

另一个边界特例：竖井洞口（Shaft Openings）类别没有宿主图元，Host 恒为 null。遍历所有洞口时，遇到竖井要么按类别跳过，要么特判处理，不能默认每个洞口都有宿主。这个"判别属性→分支读取"的模式与 FloorType.IsFoundationSlab（创建侧）、Grid.IsCurved 同族，掌握模式后可以举一反三。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 矩形洞口边界读取（3.1.7）
- **问题**: 读取墙上矩形洞口的边界顶点。
- **方法论的使用**: 先查 IsRectBoundary=true，再读 BoundaryRect 取 IList<XYZ>。
- **结论**: 矩形洞口用顶点集合表示边界。
- **结果**: 顶点可直接用于绘图/几何运算。

### 案例 2: 非矩形洞口边界（3.1.7）
- **问题**: 读取异形洞口的边界。
- **方法论的使用**: IsRectBoundary=false，读 BoundaryCurves 取 CurveArray。
- **结论**: 非矩形洞口必须走曲线分支。
- **结果**: 选错属性得 null 而非异常。

### 案例 3: 竖井洞口遍历（3.1.7）
- **问题**: 遍历所有洞口时竖井的 Host 为空。
- **方法论的使用**: 识别 Shaft Openings 类别，按类别跳过或特判。
- **结论**: 竖井洞口无宿主是对象模型边界。
- **结果**: 遍历逻辑按类别分派。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 读取墙上/板上洞口的边界顶点或曲线。
2. BoundaryRect 返回 null，怀疑代码写错。
3. 遍历文档中所有洞口，遇到竖井洞口报空引用。
4. 把洞口边界用于出图、算量、开洞复核。

### 语言信号 (用户的话里出现这些就应激活)

- "读洞口的边界" / "read the boundary of an opening"
- "BoundaryRect 怎么是 null" / "why is BoundaryRect null"
- "竖井洞口的 Host 是空的" / "shaft opening has no host"
- "IsRectBoundary" / "洞口边界是矩形还是曲线"

### 与相邻 skill 的区分

- 与 `revit-face-edge-loop-traversal`：洞口边界由 API 直接给出、走 IsRectBoundary 分支，后者是通用面边界手工遍历——读取方式对立。
- 与 `revit-new-opening-overloads`：本 skill 管读取（边界分支），后者管创建（NewOpening 重载选择）——读与写相对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **先读判别属性**
   - 完成标准: 读 Opening.IsRectBoundary，明确走哪一条分支。
   - 判停条件: 若目标是竖井洞口，跳过边界读取并按其类别特判处理。

2. **按分支取边界**
   - 完成标准: true → 读 BoundaryRect 得 IList<XYZ>；false → 读 BoundaryCurves 得 CurveArray；只读对应分支，不碰另一属性。

3. **空值防护与验证**
   - 完成标准: 对边界结果做 null 检查；遍历全部顶点/曲线验证数量与形状符合预期。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 创建洞口：那是 revit-new-opening-overloads 的职责。
- 编辑/移动洞口轮廓：本 skill 只读边界。

### 作者在书中警告的失败模式

- 先读 BoundaryRect 再查 IsRectBoundary：顺序反了就会在 null 上访问，空引用事故。
- 默认每个洞口都有宿主：竖井洞口 Host 恒为 null。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：IsRectBoundary 与两套边界属性的契约在新版本保持；竖井洞口的行为在新版仍是无宿主，但 API 对洞口的修改能力（轮廓编辑）有演进，只读场景不受影响。

### 容易混淆的邻近方法论

- BoundaryRect 的 null 与"空集合"：null 表示"该分支不适用"，不是空数据，语义不同。
- 竖井洞口与墙洞口：前者无宿主走特判，后者有宿主走常规分支。

---

## 相关 skills

- **revit-face-edge-loop-traversal**（contrasts-with）：洞口边界由 API 直接给出、走 IsRectBoundary 分支，后者是通用面边界手工遍历——读取方式对立。
- **revit-new-opening-overloads**（contrasts-with）：本 skill 管读取（边界分支），后者管创建（NewOpening 重载选择）——读与写相对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
