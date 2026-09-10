---
name: revit-element-move-decision
description: |
  移动图元按三条路径：批量/整体平移用 MoveElement/MoveElements（向量是偏移量非目标位置）；曲线驱动图元（墙/梁）"移动+改形状"用 LocationCurve.Curve；点图元（柱/房间）精确定位用 LocationPoint.Point。禁忌：基于标高图元不能跨标高移动（Z 静默忽略），被钉图元先 Pinned=false。信号："移动图元 / move element"、"移动墙改长度 / move and reshape wall"。
  Trigger: MoveElement / MoveElements / ElementTransformUtils。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.5.1 / 表 2-7 + 代码 2-31~34（约p088-091）
tags: [revit-api, move, elementtransformutils, location, geometry]
related_skills:
  - slug: revit-location-type-decision
    relation: depends-on
---

# 图元移动决策树：ElementTransformUtils vs Location 直接操作 vs LocationCurve.JoinType

## R — 原文 (Reading)

> ElementTransformUtils 类提供了两个静态方法将一个或多个图元从某处移到别处。
> MoveElement(Document, ElementId, XYZ) 根据指定向量移动文件中的一个图元。
>
> — 宦国胜, 第2章 2.5.1 / 表 2-7 + 代码 2-31、2-32、2-33、2-34（约p088-091）

---

## I — 方法论骨架 (Interpretation)

"移动图元"在 Revit 里有三条路径，按需求语义选：

1. **ElementTransformUtils.MoveElement / MoveElements**：静态方法，按**平移向量**移动一个/一组图元。要点：向量是**偏移量**（从当前位到目标位的差），不是目标坐标；保持图元形状不变。批量移动用 MoveElements（一次事务里动多图元），比循环 MoveElement 干净。
2. **LocationCurve.Curve 赋值**：曲线驱动图元（墙、梁、支撑）要"移动的同时改变长度/形状"时用——新曲线一次定义新位置+新几何。
3. **LocationPoint.Point 赋值**：点驱动图元（柱、房间、组）要精确定位时用。

两条硬禁忌：
- **基于标高的图元不能跨标高移动**——MoveElement 的 Z 分量对标高图元被静默忽略（专 skill 详述）；要改高度只能改 Base/Top Level 或偏移参数。
- **被钉住的图元（Pinned）不能移动**——先 `element.Pinned = false`。

决策顺序：先问"要不要变形"（要 → Location 路径），再问"单个还是批量"（批量 → MoveElements），最后检查标高依赖与钉住状态。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 多图元批量平移
- **问题**: 需要把一组图元整体移动。
- **方法论的使用**: 书中表 2-7 + 代码 2-31~34 用 `MoveElements(document, idCollection, translationVector)` 一次完成。
- **结论**: 批量平移的标准入口是 MoveElements，不是循环 MoveElement。
- **结果**: 一组图元按同一向量整体位移。

### 案例 2: 移动墙并改长度
- **问题**: 移动一面墙到新位置同时改变长度。
- **方法论的使用**: MoveElement 只能平移改不了形状 → 转 Location 路径，`LocationCurve.Curve = newWallLine`。
- **结论**: "平移+变形"复合需求交给 Curve 赋值。
- **结果**: 位置与长度一次更新。

### 案例 3: 点图元精确定位
- **问题**: 把柱放到精确坐标点。
- **方法论的使用**: 柱是点驱动 → `LocationPoint.Point = newLocation`。
- **结论**: 点图元定位走 Point 赋值。
- **结果**: 柱精确落位。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写任何"移动/平移图元"的命令，要选 API 入口。
2. "移动+改变长度/形状"的复合需求，发现 MoveElement 达不到效果。
3. 移动后图元位置与预期不符（尤其 Z 不对），需要排查标高约束或向量语义。
4. 批量移动多图元，纠结循环 Move 还是 MoveElements。

### 语言信号 (用户的话里出现这些就应激活)

- "移动图元 / move element / translate element"
- "移动墙并改长度 / move and resize wall"
- "批量移动 / move multiple elements at once"
- "移动向量 / translation vector / offset vector"

### 与相邻 skill 的区分

- 与 `revit-location-type-decision` 的区别: 位置类型学回答"这图元是什么驱动"；本 skill 回答"给定移动需求，三条路径走哪条"。前者是后者的分支依据。
- 与 `revit-moveelement-z-coordinate-trap` 的区别: 本决策树在禁忌处点到"标高图元不能跨标高"；Z 坐标陷阱 skill 展开静默忽略的机理与解法。
- 与 `revit-element-copy-decision` 的区别: 复制是"保留原件"的平行变换族；语义不同，API 家族相邻但选择树独立。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **解析移动语义**：纯平移？平移+变形？精确定位？批量还是单个？
   - 完成标准: 语义四问有明确答案。

2. **选路径**：平移（单个 MoveElement / 批量 MoveElements，向量=目标-当前）；变形走 LocationCurve.Curve；点定位走 LocationPoint.Point。
   - 完成标准: API 与语义匹配；向量明确标注为偏移量。
   - 判停条件: 若图元基于标高且需求含 Z 变化 → 转标高参数方案（Base/Top Level、偏移），终止移动路径。

3. **前置检查**：确认图元未被钉住（Pinned=false）；批量移动在同一事务内完成。
   - 完成标准: Pinned 检查存在；批量操作未拆散成多个事务。

4. **验证结果**：移动后读回 Location 确认位置符合预期，尤其 Z 分量。
   - 完成标准: 实际位置与预期一致，Z 静默忽略类问题被检出。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需求是复制而非移动——转复制决策树。
- 图元高度本质上由标高/偏移参数管理（柱、墙顶底）——改 Z 永远走参数，移动 API 帮不上。

### 作者在书中警告的失败模式

- MoveElement/MoveElements 不能把基于标高的图元移到标高上方或下方（如柱从 (0,0,0) 移 (10,20,30) 实际落在 (10,20,0)）。
- 被钉图元移动失败——先解锁。
- 把平移向量当目标坐标传入——结果是"从当前位置再偏移一遍"，位置错误。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：MoveElements 的参数形态与新版一致；新版补充了更多变换辅助类，但"三路径决策+标高禁忌"结构未变。

### 容易混淆的邻近方法论

- MoveElement（移动，原件消失于原位）与 CopyElement（复制，原件保留）API 形态相近，事务语义不同——选错会把"挪"变成"克隆"。

---

## 相关 skills

- revit-location-type-decision：depends-on——移动决策先判图元位置驱动类型，位置类型学提供 Curve/Point/仅基类/无 的定型依据。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
