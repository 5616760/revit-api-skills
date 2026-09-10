---
name: revit-instance-rotate-location-type
description: |
  用户要把族实例旋转任意角度、批量移动/修改点状或线状构件的位置、读 Location 时报错、或困惑 Rotate() 只转 180° 时调用。不适用于：仅翻转姿态（见 revit-instance-flip-state-check）、墙/轴网等有独立移动机制的对象。关键 trigger："旋转 45 度"、"rotate element"、"LocationPoint/LocationCurve"、"为什么 Rotate 只转一半"、"ElementTransformUtils.RotateElement"。核心：实例自带 Rotate() 仅 180°，任意角走工具类；位置按 Location 运行时类型分支读写。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.3 族实例（p161）
tags: [familyinstance, rotate, location, locationcurve, locationpoint, revit-api]
related_skills:
  - slug: revit-newfamilyinstance-overload-decision
    relation: composes-with
  - slug: revit-element-transform-utils
    relation: depends-on
---

# 族实例旋转与位置类型判断流程

## R — 原文 (Reading)

> 族实例 CanRotate 布尔属性用于检查族实例是否可被旋转 180°。如果 CanRotate 为 true，则可以调用族实例 Rotate() 方法，将族实例翻转 180°。否则，此方法不执行任何操作并返回 false。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.3 族实例（p161）

---

## I — 方法论骨架 (Interpretation)

这个 skill 是两件事的合体，都源于"API 命名/封装的陷阱"：

**第一件：旋转的两条路。**
- 实例自带的 `Rotate()` **名不副实**——它只把实例翻转 180°，且要先查 `CanRotate`，为 false 时静默返回 false。
- 要旋转任意角度（如 45°），必须改用工具类 `ElementTransformUtils.RotateElement(doc, elementId, axis, angle)`，传旋转轴（含起点）与**弧度**。

**第二件：位置信息的读写。**
- 每个图元的 `Location` 属性是多态封装，运行时是两种子类之一：
  - `LocationPoint` —— 点状构件（门、基础、家具），读写 `.Point`。
  - `LocationCurve` —— 线状构件（梁），读写 `.Curve`。
- 必须先按运行时类型分支（`is` 判断），再操作对应属性。一套代码无法通吃，因为点状和线状的"位置"语义根本不同。还有"仅 Location / 无 Location"的图元需要单独处理。

**四分支决策树**（第2章已建立）：LocationCurve / LocationPoint / 仅 Location / 无 Location。先分型，后操作。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 旋转 45°
- **问题**: 想把一个族实例旋转 45°，调用 Rotate() 结果无效。
- **方法论的使用**: 确认 Rotate() 只支持 180°，任意角改用 ElementTransformUtils.RotateElement（轴 + 弧度）。
- **结论**: "Rotate() 名不副实"是 API 命名陷阱，工具类才是通用入口。
- **结果**: 任意角度旋转成功，实例按预期摆放。

### 案例 2: 梁的位置操作
- **问题**: 创建梁后要延伸端点。
- **方法论的使用**: 梁是 LocationCurve 实例，通过 `LocationCurve.Curve` 读写曲线。
- **结论**: 线状构件的位置模型是曲线，不是点。
- **结果**: 通过曲线 API 完成端点延伸，与点状构件路径完全不同。

### 案例 3: 批量移动混合构件
- **问题**: 一批图元里有点状也有线状，如何统一移动？
- **方法论的使用**: 逐个判断 Location 运行时类型再分派读写。
- **结论**: 位置类型分支是必经步骤，无法省略。
- **结果**: 一套循环 + 分支处理两类构件。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 需要把族实例旋转任意角度，而非 180°。
2. 写"移动/对齐构件"工具，图元集合可能混合点状与线状。
3. 读 Location 时强制转换报错，怀疑类型不对。
4. 想批量延伸梁/线状族实例端点。

### 语言信号 (用户的话里出现这些就应激活)

- "旋转 45 度 / 任意角度"（"rotate by 45 degrees"）
- "为什么 Rotate() 只转 180 度"（"Rotate only flips 180 degrees"）
- "LocationPoint / LocationCurve 怎么判断"
- "移动这个构件的位置"（"move / set location of element"）
- "ElementTransformUtils.RotateElement"

### 与相邻 skill 的区分

- 与 `revit-newfamilyinstance-overload-decision`：本 skill 承接实例创建后的变换，与后者的创建重载组合成"创建→放置→旋转对齐"完整链路。
- 与 `revit-element-transform-utils`：旋转与位置读写依赖后者的 Transform 工具与约束体系。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **区分操作类型**
   - 旋转 → 步骤 2；移动/读位置 → 步骤 3。
   - 完成标准: 明确本次任务是旋转还是位置读写。

2. **旋转分支**
   - 180°：先查 `CanRotate`，true 才调 `Rotate()`；false 记录跳过。
   - 任意角：用 `ElementTransformUtils.RotateElement(doc, id, axis, angle)`，angle 用弧度。
   - 完成标准: 写出旋转代码且角度单位正确，Can\* 有守卫。

3. **位置分支**
   - 用 `is` 判断：`LocationPoint` → 读写 `.Point`；`LocationCurve` → 读写 `.Curve`；两者皆非 → 跳过或单独处理。
   - 完成标准: 每个分支都有对应类型判断，无强制转换。
   - 判停条件: 若目标是墙/轴网等非族实例，改用其专用移动 API。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需翻转姿态（朝向/把手），那属于翻转 skill 的领地。
- 墙、轴网等有独立移动机制，不走 Location 通用路径。

### 作者在书中警告的失败模式

- `Rotate()` 对任意角度**无效**（只转 180°）——想当然会用错。
- 位置属性强制转换为单一类型（如一律当 LocationPoint）会抛 InvalidCastException。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本 ElementTransformUtils 增加更多变换方法（缩放/镜像增强），但 Location 双类型模型延续。

### 容易混淆的邻近方法论

- 位置模型（Location）与几何模型（Geometry）是两套体系：Location 管"放置点/线"，Geometry 管"实体形状"。
- RotateElements（复数批量版）在 ElementTransformUtils 中与 RotateElement 成对，批量时优先复数版。

---

## 相关 skills

- **revit-newfamilyinstance-overload-decision**（composes-with）：本 skill 承接实例创建后的变换，与后者的创建重载组合成"创建→放置→旋转对齐"完整链路。
- **revit-element-transform-utils**（depends-on）：旋转与位置读写依赖后者的 Transform 工具与约束体系。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
