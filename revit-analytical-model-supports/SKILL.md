---
name: revit-analytical-model-supports
description: |
  需要查询结构图元的支撑信息（楼板/墙/柱梁/梁系统/基础的支撑来源）时调用：按图元类型选查询方向，聚合图元（BeamSystem）须先 GetBeamIds 打散到梁。Trigger：support information、IsElementFullySupported、梁系统支撑、GetSupportingElement、支撑检查工具。不适用于：分析位置几何读取、分析连接创建、非结构对象的支持关系。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.2.5（约p306-308）
tags: [revit-api, analytical-model, support, beamsystem, structure]
related_skills:
  - slug: revit-analytical-link-hub
    relation: contrasts-with
---

# 分析模型支撑（AnalyticalModelSupport）按图元类型的访问决策

## R — 原文 (Reading)

> 由于 BeamSystem 无 AnalyticalModel 属性，因此无法直接获取其支撑信息。解决方案是调用 GetBeamIds() 方法，来检索梁的 AnalyticalModelSupport 集合。
>
> — 宦国胜, 第4章 4.2.2.5（约p306–p308）

---

## I — 方法论骨架 (Interpretation)

支撑查询不是一个通用调用，而是**按图元类型选择查询方向**的决策表：

1. **楼板（Floor）**：从板自身的分析模型查 `AnalyticalModelSupports`——得到支撑它的梁/墙列表（对应 UI 草图模式里"拾取支撑"的行为）。
2. **墙**：同样从墙的分析模型查其支撑信息。
3. **柱/梁/撑（Framing）**：从构件分析模型查支撑（下方的基础或其他构件）。
4. **梁系统（BeamSystem）——反直觉点**：BeamSystem **没有** AnalyticalModel 属性，无法直接查支撑。必须先 `GetBeamIds()` 把系统打散成单根梁，再对每根梁 `GetAnalyticalModelSupports()`。
5. **基础**：独立基础用 `GetSupportingElement()` 反向拿到它支撑的族实例。
6. **通用预检**：`IsElementFullySupported()` 判断图元是否被完全支撑——"一键检查所有楼板是否完全支撑"类工具的入口。

一般规律：**系统级/聚合图元在分析层不直接暴露，需打散到 leaf 图元再查询**。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: "一键检查所有楼板是否完全支撑"
- **问题**: 要写工具检查所有楼板是否完全支撑，遇到梁系统上的板不知怎么处理。
- **方法论的使用**: 对板用 `IsElementFullySupported()` 预检 + 从板自身 AnalyticalModelSupports 查支撑列表；涉及梁系统时先 `GetBeamIds()` 打散成单根梁再逐梁查询支撑。
- **结论**: 梁系统不是独立分析载体，分析信息按成员（每根梁）分发。
- **结果**: 检查工具覆盖了直查（板/墙）与分解（梁系统）两类路径，全部楼板得到判定。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 做支撑完整性检查（悬空板/未支撑梁的查错工具）。
2. 做结构传导分析，需要"板→梁→柱→基础"的荷载路径图。
3. 用户在 BeamSystem 上找不到 AnalyticalModel / AnalyticalModelSupports，报编译错误或空引用。
4. 需要从基础反查它支撑的构件。

### 语言信号 (用户的话里出现这些就应激活)

- "支撑信息 / 是否被完全支撑" / "support information / IsElementFullySupported"
- "梁系统没有分析模型" / "BeamSystem has no AnalyticalModel"
- "GetBeamIds / 拆梁系统"
- "谁支撑这块板" / "GetSupportingElement / supporting element"

### 与相邻 skill 的区分

- 与 `revit-analytical-link-hub` 的区别：本 skill 是支撑关系的查询决策（含五方向）；该 skill 是分析连接的创建，且必须先解析 Hub 再连。
- 与 `revit-analytical-model-geometry` 的区别：该 skill 取分析位置；本 skill 查分析关系。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定图元类型并选查询方向**
   - 楼板/墙/柱梁撑 → 从自身分析模型直查；梁系统 → 走第 2 步分解；独立基础 → `GetSupportingElement()` 反查。
   - 完成标准: 每个目标图元已映射到查询方向。
   - 判停条件: 若图元是 BeamSystem，禁止直接取 AnalyticalModel——立即转入第 2 步。
2. **分解聚合图元（如遇）**
   - `beamSystem.GetBeamIds()` → 逐根梁 `GetAnalyticalModelSupports()`。
   - 完成标准: 梁系统已全部打散为单梁查询，无任何对 BeamSystem 直接访问分析模型的代码。
3. **汇总并判定**
   - 需要整体判定时用 `IsElementFullySupported()`；输出支撑列表或路径图。
   - 完成标准: 每个目标图元的支撑状态（完全支撑/部分/无）已确定并可报告。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 读取分析位置几何——那是三件套 skill 的领域。
- 创建/删除分析连接（AnalyticalLink）——支撑是只读关系，连接要 Hub 体系。
- 非结构对象（家具、幕墙嵌板等）的"支撑"概念不存在于此。

### 作者在书中警告的失败模式

- 在 BeamSystem 自身找 AnalyticalModel——不存在，编译期就找不到成员；`IsElementFullySupported` 也不适用于 BeamSystem。
- 以为系统级图元会汇总成员的支撑信息——必须逐成员查。
- 混淆"板的支撑"（板被谁撑）与"基础的支撑对象"（基础撑谁）两个方向。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版分析模型重构后支撑查询 API 有变化；"聚合图元下沉分析能力到成员"的规律依然有参考价值。
- 书中五种方向的清单基于 2014 的结构功能，新构件类型需自行扩展决策表。

### 容易混淆的邻近方法论

- BeamSystem 无分析模型反例——本决策表的特例。
- AnalyticalLink/Hub——连接（节点重合/偏移的连接）不是支撑（承载关系）。

---

## 相关 skills

- **revit-analytical-link-hub**（AnalyticalLink 永远连"中心（Hub）"而非"图元"——必须先取 Hub 再连 · contrasts-with）— 该 skill 建分析连接（且必须先取 Hub），本 skill 查询支撑关系，一建一查。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
