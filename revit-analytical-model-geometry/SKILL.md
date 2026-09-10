---
name: revit-analytical-model-geometry
description: |
  需要读取结构图元（基础/柱/梁/墙/板）的分析位置几何时调用：按 GetPoint / GetCurve / GetCurves 三件套分流，先 IsSinglePoint/IsSingleCurve 探测再调用。Trigger：analytical model position、分析模型几何、GetCurve、AnalyticalModelSelector、Raw/Active/Approximated 曲线。不适用于：支撑信息查询（走支撑决策 skill）、分析连接创建（走 Hub skill）、物理几何读取。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.2（约p302-303）
tags: [revit-api, analytical-model, geometry, structure, runtime-dispatch]
related_skills:
  - slug: revit-analytical-model-supports
    relation: contrasts-with
  - slug: revit-analytical-model-guard-clause
    relation: depends-on
---

# 分析模型（AnalyticalModel）三件套：GetPoint / GetCurve / GetCurves

## R — 原文 (Reading)

> 根据分析模型对应的图元类型，与分析相关的图元位置可以通过三种方法之一来获得：GetPoint()、GetCurve() 或 GetCurves()。请注意，从这些方法中检索的曲线不含 Reference 属性设置。
>
> — 宦国胜, 第4章 4.2.2（约p302–p303）

---

## I — 方法论骨架 (Interpretation)

分析模型把结构图元抽象为点/线/线组，读取时按"图元类型分派"：

1. **三种获取方法按类型分流**：
   - 基础（Structural Footing 等）→ `GetPoint()`（单点）；
   - 柱/梁/支撑（Framing）→ `GetCurve()`（单曲线）；
   - 墙 → `GetCurves()`（多条曲线）。
2. **必须先探测再调用**：`IsSinglePoint()` / `IsSingleCurve()` 做运行时能力探测；不判定直接调用不适配的方法会抛 `InapplicableDataException`。
3. **曲线无 Reference**：三件套返回的曲线不含 Reference——需要参照（如标注、尺寸、拾取）时，构建 `AnalyticalModelSelector` 获取曲线参照及端点，而不是直接用返回曲线。
4. **曲线语义三层**（GetCurves 接受 `AnalyticalCurveType`）：Raw（基础曲线）/ Active（屏幕可见，不含刚性连接）/ Approximated（直线段近似）；刚性连接另传 RigidLinkHead / RigidLinkTail / AllRigidLinks。
5. **工程封装**：把三分支封装成"获取分析位置"辅助方法——任何结构图元都落入三分支之一，永不崩溃。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建桁架（Truss）时取柱分析曲线端点
- **问题**: 用代码 4-13 创建 Truss 时需要柱的分析位置端点来定位桁架杆件。
- **方法论的使用**: 柱是 Framing 类图元——先 `IsSingleCurve()` 探测，再 `GetCurve()` 取分析曲线，从曲线取端点。
- **结论**: 按类型分派选对方法，一步取到端点。
- **结果**: 桁架按柱分析位置正确建立。

### 案例 2: 通用"结构图元几何提取器"
- **问题**: 输入任意结构图元，要保证不崩溃且拿到分析位置。
- **方法论的使用**: 运行时类型探测三分支：`IsSinglePoint()`→`GetPoint()`；`IsSingleCurve()`→`GetCurve()`；否则 `GetCurves()`。绝不假设所有结构图元都是曲线型或点型。
- **结论**: 三分支覆盖全部结构图元，省略预判定会在墙/基础上崩溃。
- **结果**: 提取器对异构图元集合稳定运行，无一崩溃。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 导出结构分析数据（节点坐标、杆件轴线）给外部计算程序。
2. 写通用结构图元几何提取工具，输入混合类型（基础/柱/梁/墙/板）。
3. 需要分析曲线的 Reference（标注/拾取/尺寸）——发现直接返回曲线没有 Reference。
4. 用户遇到 InapplicableDataException，想理解为什么以及对齐 Active/Raw 曲线语义。

### 语言信号 (用户的话里出现这些就应激活)

- "分析模型的位置/曲线" / "analytical model curve / GetCurve / GetCurves / GetPoint"
- "分析曲线的参照" / "AnalyticalModelSelector / reference of analytical curve"
- "InapplicableDataException" / "IsSinglePoint / IsSingleCurve"
- "Raw / Active / Approximated 曲线" / "rigid link analytical curves"

### 与相邻 skill 的区分

- 与 `revit-analytical-model-supports` 的区别：本 skill 读分析位置（GetPoint/GetCurve/GetCurves），该 skill 查分析支撑关系（谁支撑谁），一位置一关系。
- 与 `revit-analytical-model-guard-clause` 的关系：守卫 skill 是“必须先判定”的纪律本身；本 skill 是包含该守卫的三件套完整使用框架。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **取分析模型对象**
   - 结构图元 → `GetAnalyticalModel()`（注意 BeamSystem 等聚合图元没有分析模型，见支撑决策 skill）。
   - 完成标准: 拿到有效 AnalyticalModel；取不到则先分解/排除该图元。
2. **探测并分派**
   - `IsSinglePoint()` → `GetPoint()`；`IsSingleCurve()` → `GetCurve()`；否则 `GetCurves(AnalyticalCurveType)`（按 Raw/Active/Approximated 需求选择）。
   - 完成标准: 三分支返回了点/曲线/曲线组之一，过程中未调用任何未探测的方法。
   - 判停条件: 若只需要参照（Reference），跳到第 3 步用 Selector，不要用裸曲线。
3. **按需取参照或端点**
   - 构建 `AnalyticalModelSelector`（含必要信息）获取曲线参照及其端点；刚性连接用 RigidLinkHead/Tail/AllRigidLinks。
   - 完成标准: 下游（标注/导出/计算）拿到了带参照或端点的有效几何。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 查询支撑关系（楼板由谁支撑）——走分析模型支撑决策 skill。
- 创建分析连接——走 AnalyticalLink/Hub skill。
- 只要物理几何（实体、面、实体轮廓）——走 GeometryElement 体系。

### 作者在书中警告的失败模式

- 跳过 IsSinglePoint/IsSingleCurve 直接调 GetPoint/GetCurve——对墙或基础抛 InapplicableDataException（运行时崩溃级）。
- 直接用三件套返回的曲线做标注/拾取——曲线不含 Reference，必须经 AnalyticalModelSelector。
- 混淆 Raw/Active/Approximated 三层曲线语义，导出数据口径不一致。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 AnalyticalModel 已拆分为 AnalyticalNode/AnalyticalMember 等独立体系，三件套 API 被替代，本方法论的分派思想仍适用但类名需核对。
- 2014 的刚性连接（Rigid Link）在新版中模型不同。

### 容易混淆的邻近方法论

- "先判定后调用"原则 skill——本框架的前置纪律。
- Hub/AnalyticalLink——连接层，用分析模型 Id 参与，但不经三件套取几何。

---

## 相关 skills

- **revit-analytical-model-supports**（分析模型支撑（AnalyticalModelSupport）按图元类型的访问决策 · contrasts-with）— 该 skill 查“谁支撑谁”的分析关系，本 skill 取“图元在哪”的分析位置。
- **revit-analytical-model-guard-clause**（分析模型的 GetPoint / GetCurve / GetCurves 必须先判定 IsSinglePoint / IsSingleCurve · depends-on）— 调用 GetPoint/GetCurve 前必须先按 IsSinglePoint/IsSingleCurve 判定，本 skill 依赖该守卫原则。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
