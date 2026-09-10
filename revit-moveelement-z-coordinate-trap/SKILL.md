---
name: revit-moveelement-z-coordinate-trap
description: |
  MoveElement/MoveElements 对基于标高的图元（柱等）静默忽略 Z 分量：传 (10,20,30) 实际落在 (10,20,0)，不报错。柱须保持在其标高平面内，改高度只能改 Base/Top Level 参数或偏移，不能用移动向量；被钉图元先 Pinned=false。信号："移动后 Z 不对 / wrong Z after move"、"柱子垂直移动没生效 / column not moving vertically"、"MoveElement 不报错但位置不对 / silently ignored"。
  Trigger: MoveElement Z / level-based / column Z / pinned element。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.5.1（约p089）
tags: [revit-api, counter-example, moveelement, level, z-coordinate]
related_skills:
  - slug: revit-element-move-decision
    relation: depends-on
  - slug: revit-location-type-decision
    relation: depends-on
---

# MoveElement 不能跨越标高与 Z 坐标陷阱（反例单元）

## R — 原文 (Reading)

> 注意：当使用 MoveElement() 或 MoveElements() 方法时，这些方法不能将基于标高的图元移动到该标高的上方或下方。
> 例如，如果在标高 1 原点位置（0,0,0）新建一根柱，然后将其移动到新位置（10,20,30），则柱被放置在位置（10,20,0），而不是位置（10,20,30）。
>
> — 宦国胜, 第2章 2.5.1（约p089）

---

## I — 方法论骨架 (Interpretation)

这是 Revit 移动 API 的一个经典反直觉陷阱，性质是**静默失败**：

- MoveElement/MoveElements 接受 XYZ 平移向量，但对**基于标高的图元**（柱、部分墙构件等），Z 分量被**静默忽略**——不报错、不警告，图元只做平面内移动。
- 原因是模型约束：标高图元必须锚定在自己的标高平面上，Revit 用"丢弃 Z"而不是"抛异常"来维持这个不变量。
- 正确的高度操作路径是**参数而不是几何**：改 Base Level / Top Level（换锚定标高），或底部偏移 / 顶部偏移（在标高基础上抬升）。
- 相邻陷阱：被钉住的图元（Pinned = true）不能移动——先设 `element.Pinned = false` 再动。

排查思路：移动后位置不符预期且无异常 → 先查图元是否基于标高（Z 被丢）→ 再查是否被钉 → 最后才怀疑向量算错。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 柱移到 (10,20,30) 落在 (10,20,0)
- **问题**: 在标高 1 原点建柱，用 MoveElement 移到 (10,20,30)。
- **方法论的使用**: 书中以该例子直接演示陷阱——Z 分量 30 被丢弃，柱落在 (10,20,0)。
- **结论**: MoveElement 对标高图元只做平面移动，这是设计行为不是 bug。
- **结果**: 复现并确认了静默忽略语义，警示读者改 Z 必须走参数。

### 案例 2: 改高度的参数路径
- **问题**: 确实要把柱抬到 Z=30 的高度。
- **方法论的使用**: 改 Base Level/Top Level 参数或底部/顶部偏移参数，而不是移动向量。
- **结论**: 高度 = 标高参数域，位置 = 平移向量域，两域分离。
- **结果**: 通过参数操作达到目标高度。

### 案例 3: 钉住图元的移动失败
- **问题**: 移动操作对某图元无效。
- **方法论的使用**: 排查 `element.Pinned`，先置 false 再移动。
- **结论**: Pinned 是另一种"静默不动"的来源。
- **结果**: 解锁后移动成功。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. MoveElement 后图元 Z 坐标和预期不一致，且没有任何报错。
2. 需求是"把柱/标高类图元移到另一个高度"，正在考虑用移动向量实现。
3. 写批量移动脚本，部分图元没动（怀疑被钉）或动了但高度不对（标高约束）。
4. code review 时看到对柱类图元传含 Z 的移动向量。

### 语言信号 (用户的话里出现这些就应激活)

- "移动后 Z 不对 / Z ignored after MoveElement"
- "柱子移不到指定高度 / column won't move up"
- "MoveElement 不报错但没生效 / move silently fails"
- "怎么改柱的高度 / change column height / base level offset"

### 与相邻 skill 的区分

- 与 `revit-element-move-decision` 的区别: 移动决策树给出三条路径的总选择；本 skill 是其中 MoveElement 路径上的专项地雷图，聚焦 Z 静默忽略与 Pinned。
- 与 `revit-location-type-decision` 的区别: 位置类型学解释柱为什么是点驱动、标高为什么只有基类 Location；本 skill 讲移动 API 对这类图元的运行期陷阱。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **识别图元的标高依赖**：检查目标图元是否基于标高（有无 Base/Top Level 类参数）。
   - 完成标准: 标出"标高图元 / 非标高图元"。
   - 判停条件: 若非标高图元且未钉住 → Z 陷阱不适用，回到常规移动流程。

2. **拆分需求为平面移动 + 高度调整**：平面位移走 MoveElement（只用 X/Y）；高度变化走 Base/Top Level 与偏移参数。
   - 完成标准: 代码中不存在对标高图元传非零 Z 移动向量的调用；高度变化全部落在参数赋值上。

3. **移动前解锁与移动后验证**：置 `Pinned = false`；移动后读回位置验证 X/Y 生效，并确认 Z 语义由参数表达。
   - 完成标准: 验证读数与预期一致，无静默偏差残留。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 图元不基于标高（自由放置的族实例、通用模型）——Z 移动正常，不必绕参数。
- 问题根本不在移动（高度天生由参数正确管理）——直接走参数编辑。

### 作者在书中警告的失败模式

- 给标高图元传含 Z 的向量并信任结果——Z 被静默丢弃，位置错且无诊断信息。
- 忽略 Pinned 状态直接移动——操作无效。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：静默忽略语义至今如此；新版仍无警告，务必靠"读回验证"兜底。柱高度管理在新版有更多参数化选项，但"高度走参数"的分工未变。

### 容易混淆的邻近方法论

- "静默忽略"与"抛异常"是两种失败形态：MoveElement 的 Z 陷阱属于前者（最难发现），养成移动后读回验证的习惯是对策；不要因为没有异常就默认成功。

---

## 相关 skills

- revit-element-move-decision：depends-on——本 skill 是移动决策树中 MoveElement 路径上的专项地雷图，聚焦 Z 静默忽略与 Pinned。
- revit-location-type-decision：depends-on——位置类型学解释柱为何点驱动、标高为何仅基类 Location，是理解 Z 陷阱的底层依据。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
