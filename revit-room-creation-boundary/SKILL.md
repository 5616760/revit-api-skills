---
name: revit-room-creation-boundary
description: |
  需要以编程方式创建房间、放置房间到平面环、或检索房间边界段时调用。核心是两段式创建（明细表创建→PlanCircuit 放置）与"先更新后检索"的边界时序。Trigger：create room、NewRoom、房间边界、GetBoundarySegments、PlanCircuit、房间边界不更新、边界段计数不对。不适用于：族实例的房间归属（改用 ToRoom/FromRoom skill）、房间几何体积计算。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.1（约p265–p271）
tags: [revit-api, room, boundary-segments, plan-circuit, phase]
related_skills:
  - slug: revit-room-volume-enable
    relation: composes-with
---

# 房间（Room）创建与边界检索流程（含分支墙陷阱与更新时序）

## R — 原文 (Reading)

> Room 类用于表示房间和图元，如房间明细表和面积平面。…… Document.NewRoom（Phase）方法用于创建与任何指定位置无关的新房间，并将其插入现有明细表。…… 房间含有边界，它创建一个房间所在位置的封闭区域。
>
> — 宦国胜, 第4章 4.1 Revit Architecture（约p265–p271）

---

## I — 方法论骨架 (Interpretation)

房间的 API 模型分"创建"与"边界检索"两半，各有陷阱：

1. **两段式创建**：`NewRoom(phase)` 创建一个与位置无关的房间（进明细表）；真正"落地"要用 `NewRoom(room, planCircuit)` 把它放置到某平面环上。另一入口 `NewRoom(level, UV)` 直接按平面位置创建。选哪个入口取决于你是"先有房间再找位置"还是"位置已知"。
2. **边界检索**：`SpatialElement.GetBoundarySegments(SpatialElementBoundaryOptions)` 返回边界段集合，每段带 Curve 与关联图元。
3. **更新时序**：创建/修改墙等边界图元后，边界数据要等文档更新（事务提交或 Regenerate）后才反映新状态——在同一个事务内改墙后立刻读边界，读到的是旧数据。
4. **边界来源条件**：墙必须 `WALL_ATTR_ROOM_BOUNDING` 为 true 才参与边界；模型线类别须为 `OST_AreaSeparationLines`（面积分隔线）才算边界线。
5. **计数陷阱**：T 形分支墙会被 `GetBoundarySegments` 识别为两面墙（边界段数多），而 `PlanCircuit.SideNum` 视为一面——两套计数口径本就不同。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建墙后边界段"没变"的时序问题
- **问题**: 新建/修改墙之后调用 `GetBoundarySegments` 仍返回旧边界。
- **方法论的使用**: 识别为时序问题——在 Transaction 内创建/修改边界图元并提交（或 Regenerate）之后，再调用 GetBoundarySegments。
- **结论**: 边界数据是文档状态的派生数据，只反映已更新的模型。
- **结果**: 按"更新→检索"顺序调整代码后，边界正确反映新墙。

### 案例 2: 视图裁剪复用房间边界（跨章节应用）
- **问题**: 需要按房间形状裁剪视图。
- **方法论的使用**: 第 2 章视图裁剪案例中用 `room.GetBoundarySegments()` 取边界，再用 `boundarySegment.Curve` 生成裁剪形状。
- **结论**: 房间边界段是"现成的平面闭合曲线"来源，可用于任何需要房间轮廓的场景。
- **结果**: 视图裁剪形状与房间边界完全一致。

### 案例 3: 边界段计数与平面环边数不一致
- **问题**: 边界段计数与 PlanCircuit 的 SideNum 对不上，疑似代码错误。
- **方法论的使用**: 识别 T 形分支墙场景——GetBoundarySegments 把底墙算两面墙，PlanCircuit 仍算一面；同时确认 WALL_ATTR_ROOM_BOUNDING 为 true、分隔线类别为 OST_AreaSeparationLines。
- **结论**: 两者口径本不同，不是 bug；先核对边界来源条件再下结论。
- **结果**: 计数差异得到合理解释，无需"修复"。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写房间清单/面积统计工具，需要程序化创建房间并放置到位置。
2. 需要按房间边界生成轮廓（视图裁剪、面层布置、净高分析等）。
3. 用户困惑"改了墙但房间边界没更新""边界段数量和墙数对不上"。
4. 需要按 Phase 创建房间或按平面环（PlanCircuit）批量放置房间。

### 语言信号 (用户的话里出现这些就应激活)

- "创建房间 / 放置房间" / "create / place room, NewRoom"
- "房间边界段" / "GetBoundarySegments / boundary segments"
- "边界没更新 / 还是旧边界" / "room boundary not updating"
- "PlanCircuit / 平面环 / SideNum"

### 与相邻 skill 的区分

- 与 `revit-room-volume-enable` 的关系：本 skill 处理房间创建与边界段检索（平面问题）；该 skill 处理体积计算必须显式启用的开关问题（三维问题），组合覆盖房间处理的完整链路。
- 与 `revit-to-from-room-semantics` 的区别：该 skill 解决门窗与房间的归属判断，依赖房间已存在；本 skill 负责房间本身的创建与边界。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **选择创建入口**
   - 位置已知 → `NewRoom(level, UV)`；先有房间（如按 Phase 建明细表）→ `NewRoom(phase)` + `NewRoom(room, planCircuit)` 放置。
   - 完成标准: 房间已创建并（如需）放置到目标位置。
2. **确保模型已更新**
   - 边界图元（墙/分隔线）的创建/修改在事务内完成后，先提交（或 Regenerate）再读边界。
   - 完成标准: 检索边界前文档状态已包含全部边界图元变更。
   - 判停条件: 若只做创建不读边界，跳到第 4 步。
3. **检索并核对边界**
   - `GetBoundarySegments(SpatialElementBoundaryOptions)` 取段；核对 `WALL_ATTR_ROOM_BOUNDING` 与分隔线类别（OST_AreaSeparationLines）。
   - 完成标准: 边界段集合与预期房间形状一致；若计数异常，按"分支墙口径差"解释而非报错。
4. **消费边界曲线**
   - 每段取 `boundarySegment.Curve` 用于裁剪/布置/统计。
   - 完成标准: 下游几何操作拿到有效曲线集合。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 求族实例（家具/设备）属于哪个房间——那是 FamilyInstance.Room / ToRoom/FromRoom 的领域。
- 计算房间体积——体积有独立的全局设置开关（见体积计算启用 skill）。
- 未放置（Unplaced）房间的处理不是本 skill 重点，但注意其没有边界可读。

### 作者在书中警告的失败模式

- 同一事务内改墙后立即读边界——读到旧数据，必须先提交/Regenerate。
- 忽略 `WALL_ATTR_ROOM_BOUNDING`，以为所有墙都构成边界。
- 把 GetBoundarySegments 与 PlanCircuit.SideNum 的计数差异当 bug 修——分支墙口径本不同。
- 非 OST_AreaSeparationLines 类别的模型线不参与边界。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 GetBoundarySegments 的签名（返回类型、选项参数）有变化，SpatialElementBoundaryOptions 的细项也有扩充。
- 2014 的 Phase/阶段工作流描述较简略，多阶段翻新项目的房间归属需额外小心。

### 容易混淆的邻近方法论

- ToRoom/FromRoom 语义——那是"门窗↔房间"关系；本 skill 是"房间↔边界图元"关系。
- SpatialElementGeometryCalculator——计算房间三维几何，比边界段更进一步。

---

## 相关 skills

- **revit-room-volume-enable**（房间体积计算必须显式启用（VolumeComputationEnable） · composes-with）— 房间体积计算需显式启用 VolumeComputationEnable，本 skill 创建房间并给出边界，体积读取由该 skill 补齐。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
