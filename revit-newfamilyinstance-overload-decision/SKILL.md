---
name: revit-newfamilyinstance-overload-decision
description: |
  用户要用 NewFamilyInstance 放置构件但不确定用哪个重载、实例 3D 显示异常、或报"符号未加载"时调用。不适用于：批量放置同类型实例（见 revit-newfamilyinstances-batch-create）。关键 trigger："放置门/梁/灯具用哪个重载"、"which overload"、"实例 3D 里显示不对"、"基于面的族怎么放"。核心：12 个重载按"类别+位置特征（host/Level/Face/Reference/线性/View）"五维选择；重载选错是静默显示错误而非异常。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.3 族实例（p163）
tags: [newfamilyinstance, overload, familyplacement, based-on-face, revit-api]
related_skills:
  - slug: revit-newfamilyinstances-batch-create
    relation: contrasts-with
  - slug: revit-family-symbol-instance-navigation
    relation: composes-with
  - slug: revit-loadfamilysymbol-preference
    relation: composes-with
  - slug: revit-instance-host-subcomponent-navigation
    relation: composes-with
---

# NewFamilyInstance 重载选择决策

## R — 原文 (Reading)

> 方法具有 12 个重载方法。选择使用哪个重载不仅取决于实例的类别，而且与其他位置特征也有关，如该实例对象是否应嵌入主体、置于相关参照标高，或直接置于特定的面。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.3 族实例（p163）

---

## I — 方法论骨架 (Interpretation)

`NewFamilyInstance` 有 12 个重载，不能靠"猜哪个能编译过"来选——要按**五维决策空间**来选：

1. **是否需要宿主（Host）**：门/窗要嵌入墙 → 带 host 的重载；桌子/家具自由放 → 无 host 重载。
2. **是否需要标高（Level）**：基于标高的构件 → 带 level；否则**不要带**，带错会出现"静默的显示错误"。
3. **是否需要面（Face/Reference）**：基于面的灯具/幕墙嵌板 → 带 face 或 reference。
4. **是否线性**：梁 → 用 Curve 重载（沿曲线放置）。
5. **是否依赖视图（View）**：少数重载需要视图上下文。

典型映射（书中四个创建案例）：
- 桌子 → 无 host、无 level 的重载。
- 梁 → Curve 重载。
- 门 → wall（host）+ level 重载。
- 床 → direction + host 重载（有方向与主体）。

**两类失败模式要分清**：符号未加载 → 抛异常；重载选错 → 不抛异常，只是 3D 视图显示不对（静默）。所以排查方向是"重载维度"而非"几何参数"。创建前先确认 FamilySymbol 已 LoadFamily/LoadFamilySymbol 加载，是硬性前提。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 桌子放置与 Level 陷阱
- **问题**: 桌子用带 Level 的重载后，3D 视图显示异常。
- **方法论的使用**: 桌子不是基于标高的构件，正确选择"无 level"的重载。
- **结论**: 重载选择错误不报错，只表现为显示问题。
- **结果**: 换对重载后 3D 显示正常。

### 案例 2: 梁用 Curve 重载
- **问题**: 创建梁，放置位置沿一条线。
- **方法论的使用**: 梁是线性构件，走 Curve 版 NewFamilyInstance。
- **结论**: "是否线性"是选重载的关键维度。
- **结果**: 梁沿曲线正确放置，且可用 LocationCurve 继续操作。

### 案例 3: 门必须给墙主体与标高
- **问题**: 创建门需要正确嵌入墙。
- **方法论的使用**: 用带 wall + level 的重载。
- **结论**: 门窗是"嵌入主体+基于标高"双维度构件。
- **结果**: 门正确开洞、随墙移动。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写放置代码，面对 12 个重载不知道选哪个。
2. 实例创建成功但 3D 视图显示异常——先怀疑重载维度。
3. 创建报异常说符号未加载/未找到。
4. 要放置基于面的灯具、幕墙嵌板等特殊类别。

### 语言信号 (用户的话里出现这些就应激活)

- "放置门/梁/桌子/灯具用哪个重载"（"which overload for placing a door/beam"）
- "实例在 3D 视图显示不对"（"instance not displayed correctly in 3D"）
- "基于面的族怎么放置"（"place family on a face"）
- "NewFamilyInstance 符号未加载"（"symbol not loaded"）
- "NewFamilyInstance 参数/重载"（"NewFamilyInstance overloads"）

### 与相邻 skill 的区分

- 与 `revit-newfamilyinstances-batch-create`：本 skill 管单个实例选重载，后者管 N 个同型实例批量创建——数量与参数结构决定走哪条。
- 与 `revit-family-symbol-instance-navigation`：组合其"取符号"能力作为创建前置，再选对重载放置。
- 与 `revit-loadfamilysymbol-preference`：组合其加载策略，先加载符号再放置实例。
- 与 `revit-instance-host-subcomponent-navigation`：组合其宿主/子构件导航，以处理带主体的放置重载。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **收集五维答案**
   - 逐项回答：要宿主？要标高？放面上（face/reference）？线性？需视图？类别是什么？
   - 完成标准: 得到明确的五维向量（每个问题 true/false）。

2. **按向量对照 12 重载签名**
   - 从签名中挑出同时满足所有维度的唯一重载；拿不准时先对照四个案例映射。
   - 完成标准: 写出所选重载的完整参数列表，与向量一一对应。

3. **前置检查与失败诊断**
   - 调用前确认 FamilySymbol 已加载（未加载会抛异常）。
   - 调用后若显示异常，**先复查重载维度**再查几何参数。
   - 完成标准: 实例放置成功且 3D 显示正确。
   - 判停条件: 若要放置多个同类型实例，改用 `NewFamilyInstances` 批量接口。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要批量创建同类型实例 → NewFamilyInstances（revit-newfamilyinstances-batch-create）。
- 创建非族实例（墙/轴网/标注），各有专用方法。

### 作者在书中警告的失败模式

- 符号未加载 → 抛异常；**重载选错 → 静默显示错误**（不抛异常，最难排查）。
- 桌子这类自由构件误带 Level 参数 → 3D 显示异常，排查方向应回到重载维度。

### 作者的盲点 / 时代局限

- 基于 Revit 2014 的 12 个重载：后续版本新增重载（如 Face+Reference、按平面放置等），但五维决策逻辑依然适用。

### 容易混淆的邻近方法论

- NewFamilyInstance（单数）与 NewFamilyInstances（复数）是两条 API 通道，别混用。
- 创建时的"位置特征"（host/level/face）与创建后改 Location 是两回事：前者选重载，后者改姿态。

---

## 相关 skills

- **revit-newfamilyinstances-batch-create**（contrasts-with）：本 skill 管单个实例选重载，后者管 N 个同型实例批量创建——数量与参数结构决定走哪条。
- **revit-family-symbol-instance-navigation**（composes-with）：组合其"取符号"能力作为创建前置，再选对重载放置。
- **revit-loadfamilysymbol-preference**（composes-with）：组合其加载策略，先加载符号再放置实例。
- **revit-instance-host-subcomponent-navigation**（composes-with）：组合其宿主/子构件导航，以处理带主体的放置重载。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
