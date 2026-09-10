---
name: revit-location-type-decision
description: |
  操作图元位置前先判 Location 四分类型：LocationCurve（墙/梁/支撑，设 Curve 同改位置与长度）、LocationPoint（房间/柱/组，设 Point）、仅基类 Location（标高/楼板，不能定位）、无 Location（视图/线荷载）。用前须向下转型。信号："移动墙并改长度 / move wall change length"、"柱子定位 / locate column"、"element location"。不适用：批量平移（MoveElement）与跨标高移动。
  Trigger: LocationCurve / LocationPoint / element location。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.5.4 / 表 1-15（约p075）
tags: [revit-api, location, geometry, element-position, decision-framework]
related_skills: []
---

# 图元位置类型决策：LocationCurve / LocationPoint / 仅 Location / 无 Location

## R — 原文 (Reading)

> 墙、梁和支撑是曲线驱动的，使用 LocationCurve。房间、房间标记、高程点尺寸、组、非曲线驱动的族实例以及所有内建族实例，使用 LocationPoint。
> Level、Floor、BeamSystem、Rebar、Reinforcement…… 仅 Location。
>
> — 宦国胜, 第1章 1.5.4 / 表 1-15（约p075）

---

## I — 方法论骨架 (Interpretation)

Revit 图元的"位置"不是统一属性，而是按驱动方式分四种，操作前必须先定型：

1. **LocationCurve（曲线驱动）**：墙、梁、支撑等。位置 = 一条曲线（Line/Arc）。直接给 `LocationCurve.Curve` 赋新曲线，可以**同时**改位置、长度甚至形状（直墙变斜墙）——这是 MoveElement 做不到的。
2. **LocationPoint（点驱动）**：房间、房间标记、组、非曲线驱动的族实例（柱等）。位置 = 一个 XYZ 点，赋值 `LocationPoint.Point` 即移动定位点。
3. **仅 Location（基类）**：标高、楼板、部分标记、钢筋、荷载等。只有基类 Location，向下转型会失败，不能用 Curve/Point 方式定位——要改位置得走参数或专有 API。
4. **无 Location**：视图、线荷载、边界条件等，根本没这个属性。

`Element.Location` 返回的是抽象基类，**必须向下转型**（as LocationCurve / as LocationPoint）才能拿到具体几何。判断流程：取 Location → 判空 → 尝试转型 Curve → 尝试转型 Point → 都失败即属后两类。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 移动墙并改变长度
- **问题**: 要把一面墙移到新位置并改变长度。
- **方法论的使用**: 墙是曲线驱动 → `LocationCurve.Curve = newWallLine`，一次赋值同时移动+变形。MoveElement 只能平移，改不了形状。
- **结论**: "移动+变形"需求必须走 LocationCurve，不走 MoveElement。
- **结果**: 位置和长度一次到位。

### 案例 2: 点图元定位
- **问题**: 移动柱/房间到新位置。
- **方法论的使用**: 点驱动 → `LocationPoint.Point = newXYZ`。
- **结论**: 点图元用 Point 赋值，语义清晰。
- **结果**: 柱/房间精确定位到目标点。

### 案例 3: 旋转能力依赖类型
- **问题**: 判断某族实例能否旋转、怎么旋转。
- **方法论的使用**: FamilyInstance 的旋转能力依赖其 Location 类型——先定型再谈旋转 API。
- **结论**: Location 类型决定可用的几何操作集合。
- **结果**: 旋转路径按类型选择，避免对不支持的类型硬调。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要"移动并改变形状/长度"的复合需求（MoveElement 平移不够用）。
2. 对 Location 直接调用发现它是抽象的、拿不到坐标——缺向下转型。
3. 对标高、楼板、视图这类图元取 Location 失败或转型失败——本就属"仅 Location/无 Location"类。
4. 写通用几何处理代码，需要先识别图元的位置驱动方式再分支。

### 语言信号 (用户的话里出现这些就应激活)

- "移动墙并改长度 / move wall and change length"
- "图元位置 / element location / get position"
- "LocationCurve LocationPoint 区别 / location cast"
- "柱/房间定位点 / locate column / room position"

### 与相邻 skill 的区分

- 与 `revit-element-move-decision` 的区别: 移动决策树裁决"MoveElement vs Location vs 其他"的整体路径；本 skill 是 Location 分支内部"Curve/Point/基类/无"的类型学基础。
- 与 `revit-moveelement-z-coordinate-trap` 的区别: Z 陷阱讲 MoveElement 对标高图元的静默忽略；本 skill 解释为什么标高图元本就属"仅 Location"，改高度要走 Level 参数。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **取 Location 并判空**：`element.Location`，为 null 即"无 Location"类（视图、线荷载等），位置操作改走参数/专有 API。
   - 完成标准: null 情况显式处理，不空引用。

2. **向下转型定型**：依次尝试 `as LocationCurve`、`as LocationPoint`。
   - 完成标准: 类型确定为 Curve / Point / 仅基类三态之一。
   - 判停条件: 两个转型都失败 → 属"仅 Location"类（标高/楼板/钢筋等），跳到步骤 4 的替代路径。

3. **按型操作**：Curve 型赋新曲线（位置+形状一次改）；Point 型赋新点（仅移动定位点）。
   - 完成标准: 赋值语句与驱动类型匹配。

4. **替代路径**：仅基类/无 Location 的图元，位置语义走对应参数（如 Level、偏移）或专有 API。
   - 完成标准: 替代路径明确，无"对基类 Location 硬取坐标"的代码。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 批量整体平移多个图元——ElementTransformUtils.MoveElements 更合适（见移动决策树）。
- 图元被钉住（Pinned）——先解锁再动位置，位置 API 类型学帮不了这个。

### 作者在书中警告的失败模式

- 忘记向下转型直接用 Location——拿不到任何具体几何。
- 对"仅 Location"类图元强行转型 Curve/Point——转型失败，位置操作落空。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：表 1-15 的类型归属在新版基本稳定，但新图元类型加入后归属需查证；部分新版 API 提供了更多定位辅助（如 Element.get_BoundingBox 不变，仍非位置语义）。

### 容易混淆的邻近方法论

- Location（定位驱动）与 BoundingBox（显示包围盒）都能"描述位置"，但前者是可写的驱动几何、后者是只读的近似包围——不要用 BoundingBox 去改位置。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
