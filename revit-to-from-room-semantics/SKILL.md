---
name: revit-to-from-room-semantics
description: |
  需要判断门/窗两侧各属于哪个房间、或用门推导房间邻接关系（连通图）时调用。核心是 ToRoom/FromRoom 双向动态属性与判空。Trigger：ToRoom / FromRoom、门开向哪个房间、相邻房间关系图、room adjacency graph、FlipFromToRoom。不适用于：普通族实例的房间归属（用 Room 属性）、房间边界检索、非门窗图元的相邻推导（需几何方法）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.1.7（约p272-273）
tags: [revit-api, room, door, family-instance, adjacency]
related_skills:
  - slug: revit-room-creation-boundary
    relation: depends-on
---

# 门/窗的 ToRoom 与 FromRoom 语义及房间归属判断

## R — 原文 (Reading)

> 在 API 中（也只有在 API 中），Door 图元有两个附加属性，其参考门的相反两个面域：ToRoom 和 FromRoom。若区域为房间，则该属性的值将是一个 Room 图元。若区域非房间，则该属性返回 null。
>
> — 宦国胜, 第4章 4.1.7 房间和族实例（约p272–p273）

---

## I — 方法论骨架 (Interpretation)

这是 API 独有（UI 里看不到）的一对门窗专属属性，语义固定：

1. **方向语义**：门开向一侧的区域是 `ToRoom`，另一侧是 `FromRoom`——即"从哪个房间来（From）、到哪个房间去（To）"。
2. **返回值语义**：值是 Room 图元；若该侧区域不是房间（走廊未封闭、外部空间等），返回 null；两侧可以同时为 null（门悬在无房间区域）。
3. **动态性**：随区域变动动态更新——重新划分房间后无需重新计算。
4. **归属层级**：所有 FamilyInstance 都有 `Room`（所在房间），但**只有门/窗**有 FromRoom/ToRoom——这是 API 特设。
5. **典型应用——房间邻接图**：遍历项目中的门，两侧都非 null 则建立"两个房间通过该门连通"的边；`FlipFromToRoom()` 可交换朝向做验证。

核心思想：把门当作房间图的"边"、房间当"节点"，用 API 现成的双向句柄免掉几何求交。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 自动生成相邻房间关系图
- **问题**: 用 API 生成"哪两个房间通过一扇门连通"的邻接图。
- **方法论的使用**: 门是天然的房间连接器——遍历 FamilyInstance，对 Door/Window 读 FromRoom/ToRoom，两侧非 null 则建立连通边。
- **结论**: 无需任何几何求交即可构建房间邻接图。
- **结果**: 自动产出房间连通关系，可用 `FlipFromToRoom()` 交换朝向再验证门开向。

### 案例 2: 判空的必要性
- **问题**: 门窗悬在未封闭区域或外部空间时读这两个属性。
- **方法论的使用**: 按语义先判空——外部房间（无封闭空间）对应属性为 null，甚至两侧同时 null。
- **结论**: 读 FromRoom/ToRoom 前必须判空，否则 NullReferenceException。
- **结果**: 带判空的遍历对所有门窗安全运行。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 做逃生路径/人流分析，需要知道哪些房间互相连通（经哪扇门）。
2. 做门窗清单时要标注"门开向哪个房间、从哪个房间来"。
3. 需要验证门的朝向是否合理，想程序化翻转（FlipFromToRoom）再比较。
4. 用户问"为什么这个门的 ToRoom 是 null"或"怎么判断门两侧各是什么房间"。

### 语言信号 (用户的话里出现这些就应激活)

- "门开向哪个房间 / 两侧是什么房间" / "which room does the door open into / ToRoom / FromRoom"
- "房间连通关系 / 邻接图" / "room adjacency graph / connected rooms"
- "翻转门的朝向" / "FlipFromToRoom"
- "门窗与房间的关系" / "door room assignment"

### 与相邻 skill 的区分

- 与 `revit-room-creation-boundary` 的关系：本 skill 做门窗与房间的归属判断（FromRoom/ToRoom），依赖房间已存在；该 skill 负责房间自身的创建与边界检索，二者是依赖关系。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **收集门窗集合**
   - 过滤 `FamilyInstance` 且类别为 Door/Window。
   - 完成标准: 得到门窗实例清单；若目标是邻接图，同时收集全部 Room 节点。
   - 判停条件: 若只关心普通族实例所在房间，直接读 `FamilyInstance.Room` 即可，不走本 skill 的双属性。
2. **读取并判空**
   - 逐个读 `FromRoom` / `ToRoom`，两侧分别判 null。
   - 完成标准: 每个门窗的两侧归属已确定（Room 或明确记录为 null 及原因）。
3. **构建关系并验证**
   - 邻接图：两侧非 null 则加边；朝向验证：`FlipFromToRoom()` 交换后再读对比。
   - 完成标准: 邻接边集合/朝向结论已产出且与模型当前状态一致（属性随区域动态更新，无需缓存）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 普通族实例（家具、设备）的房间归属——没有 FromRoom/ToRoom，只有 `Room` 属性。
- 想推导**非门窗**图元的相邻房间关系——这两个属性不开放，只能用几何方法（求交/距离）。
- 房间边界/轮廓问题——走边界检索 skill。

### 作者在书中警告的失败模式

- 不判空直接使用返回值——外部空间或未封闭区域两侧可为 null、甚至同时 null。
- 期待 UI 里有这个信息——ToRoom/FromRoom 是"只有 API"才暴露的属性。
- 以为属性需要手动刷新——它们随区域变动动态更新，缓存反而拿到过期数据。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 Phase（阶段）对 Room 归属的影响更复杂（FromRoom/ToRoom 带 Phase 语义），本书未展开。
- 门"开向侧=ToRoom"的语义在极个别开门方向配置下易误读，建议结合 Flip 验证。

### 容易混淆的邻近方法论

- `FamilyInstance.Room`（所在房间）——单向、全族实例可用；FromRoom/ToRoom 是双向、仅门窗。
- 房间边界检索——回答"边界在哪"，本 skill 回答"经什么连通"。

---

## 相关 skills

- **revit-room-creation-boundary**（房间（Room）创建与边界检索流程（含分支墙陷阱与更新时序） · depends-on）— 门窗↔房间的归属判断以房间已创建为前提，本 skill 依赖该 skill 的房间创建与边界能力。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
