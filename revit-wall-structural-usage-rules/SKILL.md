---
name: revit-wall-structural-usage-rules
description: |
  创建墙时关心结构用法与房间边界如何被推导时。API 创建墙遵循隐式推导：结构参数 true 或 WallType 功能为 Foundation → 结构用法为 Bearing，否则 NonBearing（OR 关系）；WallType 功能为 Retaining → Room Bounding=false。创建承重/挡土墙前必须先校验 WallType 功能属性，否则得错误行为。不适用于：墙几何提取、定位线计算。
  Trigger："structural=false 却得到承重墙"、"怎么让墙不参与房间边界"、"创建挡土墙"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.1 墙（p146）
tags: [wall, structural-usage, room-bounding, implicit-derivation, walltype]
related_skills: []
---

# 墙结构用法与房间边界参数的推导规则

## R — 原文 (Reading)

> API 创建的墙遵循：输入结构参数为 true 或墙功能为 Foundation（基础），则墙结构用法为 Bearing（承重）；否则为 NonBearing（非承重）。墙功能为 Retaining（挡土墙）时，Wall 的 Room Bounding 为 false。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.1 节墙（p146）

---

## I — 方法论骨架 (Interpretation)

API 创建墙时，有些参数不是你传了才有的，而是按既定规则"推导"出来的——两个结构相关参数尤其如此：

结构用法（Structural Usage）：你显式传 structural=true，或者所选 WallType 的墙功能（WALL_ATTR_EXTERIOR）是 Foundation（基础），两者任一成立，创建出的墙结构用法就是 Bearing（承重）；否则是 NonBearing。注意这是 OR 关系——哪怕你传了 structural=false，只要类型功能是基础，结果仍是承重墙。

房间边界（Room Bounding）：如果 WallType 墙功能是 Retaining（挡土墙），创建出的墙 Room Bounding 自动为 false（不参与房间边界）。

这条原则的实操含义：创建承重墙或挡土墙之前，必须先读并校验 WallType 的功能属性；否则会在不知情的情况下拿到错误行为——想建非承重墙却因为类型是基础类型而得到承重墙，这类问题不报错、极难发现。它背后是更宽的模式：Revit API 创建的图元参数不都由调用者显式指定，部分由类型属性隐式推导。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 承重推导（3.1.1）
- **问题**: 需要确认创建的墙是承重还是非承重。
- **方法论的使用**: 按规则查 structural 参数与 WallType 功能的 OR 组合。
- **结论**: Foundation 类型功能可单独触发承重。
- **结果**: structural=false 也可能得到 Bearing。

### 案例 2: 挡土墙与房间边界（3.1.1）
- **问题**: 想让墙不参与房间边界。
- **方法论的使用**: 用 Retaining 功能的墙类型，Room Bounding 自动 false。
- **结论**: 推导规则提供一条隐式路径。
- **结果**: 也可显式改 WALL_ATTR_ROOM_BOUNDING。

### 案例 3: 上下文推导模式的复现（3.2.4、3.5.3）
- **问题**: 理解"参数由环境推导"是普遍模式。
- **方法论的使用**: NewFamilyInstance 用位置的标高而非层标高；Created Phase 默认与当前视图一致。
- **结论**: 推导不是墙独有，是 API 通用契约。
- **结果**: 见 s2b-c04、s2b-t09。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 创建墙后结构用法与预期不符（想非承重却得承重）。
2. 需要让墙不参与房间边界（或用 Retaining 类型隐式实现）。
3. 写批量建墙工具，需预判每个墙的结构属性。
4. 排查"为什么这堵墙在房间面积里没被算作边界"。

### 语言信号 (用户的话里出现这些就应激活)

- "structural=false 怎么还是承重墙" / "wall is bearing even though structural is false"
- "怎么让墙不参与房间边界" / "make a wall not room bounding"
- "创建挡土墙" / "create a retaining wall"
- "墙结构用法怎么推导的" / "how is wall structural usage derived"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **创建前校验类型功能**
   - 完成标准: 读取将要使用的 WallType 墙功能（WALL_ATTR_EXTERIOR），确认是 Foundation 还是 Retaining，并推演最终会得到什么结构用法/房间边界。

2. **按预期选参数**
   - 完成标准: 需要非承重墙时确认类型功能非 Foundation；需要挡土效果时选 Retaining 类型；不需要挡土时避开。
   - 判停条件: 若发现类型功能会推导出非预期行为，返回步骤 1 换类型，不要继续创建。

3. **创建后复核**
   - 完成标准: 创建后读回结构用法与 Room Bounding，与预期一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只是创建普通墙、不关心结构属性：走 revit-wall-create-overloads。
- 修改已有墙的定位线/几何：与本 skill 无关。

### 作者在书中警告的失败模式

- 以为 structural=false 就能保证非承重——类型功能 Foundation 会单独触发承重。
- 不知道 Retaining 自动关房间边界：创建挡土墙后房间统计"缺了边界"却查不到原因。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：墙功能枚举与推导规则在后续版本保持；但新版本中结构墙的展示与设置路径有 UI 变化，API 契约仍以此为准。

### 容易混淆的邻近方法论

- "显式传参"与"隐式推导"：有的参数你传了才算，有的参数由类型推导覆盖——先分清哪类再排错。
- 结构用法（Structural Usage）与结构类型（Structural Type）：前者是墙的参数，后者是构件分类，不同维度。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
