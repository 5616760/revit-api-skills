---
name: revit-new-opening-overloads
description: |
  创建洞口选 NewOpening 重载时。入口由"主体类型"决定，4 种重载：族实例 → NewOpening(Element, CurveArray, eRefFace)，iFace 指定参照面；屋顶/楼板/天花板 → 板类重载；竖井 → 双标高重载；墙 → NewOpening(Wall, XYZ, XYZ) 两点矩形。能力边界：墙上只支持矩形洞口，弧形墙非矩形洞口改走墙轮廓编辑。不适用于：读洞口边界（revit-opening-boundary-reading）、墙定位线。
  Trigger："在梁上开洞"、"怎么创建洞口"、"弧形墙上开洞"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.7 洞口（p155）
tags: [opening, new-opening, overload-selection, host-type, wall-opening]
related_skills:
  - slug: revit-new-opening-constraints
    relation: composes-with
---

# NewOpening 重载方法选择决策

## R — 原文 (Reading)

> 在 Revit 平台 API 中，使用 Document.NewOpening() 方法可在项目中创建洞口。不同的主体图元上，可以通过代码 3-7 所列的四种方法重载来创建洞口。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.7 节洞口（p155）

---

## I — 方法论骨架 (Interpretation)

NewOpening 不是"一个开洞函数"，而是按主体类型切成的四个重载——先问"在什么上面开洞"，再选签名：

族实例（梁、柱、撑等）→ NewOpening(Element, CurveArray, eRefFace)，第三个参数指定洞口开在主体的哪个参照面上，弧形面也没问题；屋顶/楼板/天花板 → 板类重载，通常带"是否垂直"开关；竖井 → 双标高重载（topLevel 与 bottomLevel 界定竖向范围）；墙 → NewOpening(Wall, XYZ, XYZ)，两个点构成矩形角点。

重载之外还有一条跨 API 的能力边界：墙上只支持矩形洞口。想在弧形墙上开非矩形洞口，Opening API 直接无能为力，必须改走"编辑墙轮廓"的路径。决策心法与 Wall.Create、NewFamilyInstance 同一哲学：按主体/参数形态选重载，而不是按"想要的效果"找函数。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 族实例上开洞（3.1.7 代码 3-7）
- **问题**: 在梁上开洞口。
- **方法论的使用**: NewOpening(Element, CurveArray, eRefFace)，用 iFace 指定开在哪个参照面。
- **结论**: 开洞位置由参照面参数显式控制。
- **结果**: 弧形参照面也可用。

### 案例 2: 墙上开矩形洞口（3.1.7 代码 3-7）
- **问题**: 在直墙/弧墙上开矩形洞口。
- **方法论的使用**: NewOpening(Wall, pntStart, pntEnd) 两点矩形重载。
- **结论**: 墙洞口只有矩形一种形态。
- **结果**: 两点必须构成矩形角点（见 revit-new-opening-constraints）。

### 案例 3: 非矩形洞口的能力边界（3.1.7）
- **问题**: 需要墙上的非矩形洞口。
- **方法论的使用**: 确认 Opening API 不支持，改走编辑墙轮廓路径。
- **结论**: 这是跨 API 的能力边界。
- **结果**: 需求驱动的路径迁移。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在梁/柱/撑等族实例上开洞。
2. 在屋顶、楼板、天花板上开洞。
3. 创建竖井洞口。
4. 在墙上开矩形洞口；或要非矩形洞口时判断能力边界。

### 语言信号 (用户的话里出现这些就应激活)

- "在梁上开一个洞" / "create an opening in the beam"
- "怎么在墙上开洞口" / "create a wall opening"
- "弧形墙上能开非矩形洞口吗" / "non-rectangular opening on a curved wall"
- "NewOpening 有哪几个重载" / "NewOpening overloads"

### 与相邻 skill 的区分

- 与 `revit-new-opening-constraints`：本 skill 管重载选型，后者管参数约束与异常条件，组合成完整洞口创建流程。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **识别主体类型**
   - 完成标准: 明确洞口开在 族实例 / 板类 / 竖井 / 墙 中的哪一类，对应到四个重载之一。
   - 判停条件: 若目标是在墙上开非矩形洞口，判停——改走编辑墙轮廓流程。

2. **准备参数并调用**
   - 完成标准: 按重载准备参数：族实例→Element + CurveArray + eRefFace；墙→两点矩形角点；竖井→两个标高（校验上下顺序）。
   - 判停条件: 竖井调用前先校验 topLevel > bottomLevel，否则跳到 revit-new-opening-constraints 的验证流程。

3. **验证洞口**
   - 完成标准: NewOpening 返回非 null 的 Opening；洞口位置、形状与预期一致；无异常。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只是读取已有洞口边界：走 revit-opening-boundary-reading。
- 修改墙轮廓实现非矩形开洞：那是墙轮廓编辑，不是 Opening API。

### 作者在书中警告的失败模式

- 墙上开洞只有矩形：非矩形需求直接撞能力墙。
- 重载按主体类型选，选错重载（如拿族实例重载传 Wall）编译或运行失败。
- 竖井 topLevel 必须高于 bottomLevel，否则抛异常。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：4 个重载的清单在后续版本保持；新版对洞口轮廓编辑（修改洞口边界形状）能力有增强，若需求升级可查新 API，但本 skill 的能力边界结论仍成立。

### 容易混淆的邻近方法论

- 重载选择 vs 能力边界：重载覆盖"四类主体"，非矩形墙洞属于"API 外"路径，不是第五个重载。
- eRefFace 参照面 vs 图元表面：前者是几何参照，后者是图元概念。

---

## 相关 skills

- **revit-new-opening-constraints**（composes-with）：本 skill 管重载选型，后者管参数约束与异常条件，组合成完整洞口创建流程。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
