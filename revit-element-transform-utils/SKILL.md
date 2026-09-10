---
name: revit-element-transform-utils
description: |
  Revit 构件几何变换（旋转/镜像/阵列/对齐/成组/删除/固定）不知调哪个 API；变换"不生效"排障（Pinned 不能旋转、Move/Rotate 返回 true 不动、镜像需 CanMirror）；删除要级联清单；关联阵列成员随源更新。不适用：纯移动/复制（revit-element-move-decision/revit-element-copy-decision）、族实例翻转（revit-instance-flip-state-check）。
  Trigger："旋转/镜像/阵列/对齐"、"rotate/mirror/array"、"为什么旋转没反应"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.5 节编辑图元（约 p108–p113）
tags: [element-transform, elementtransformutils, api-selection, rotate, array]
related_skills: []
---

# 图元几何变换 API 选择框架

## R — 原文 (Reading)

> ElementTransformUtils 类提供两种静态方法，用于旋转或镜像图元……NewAlignment() 可在两个参照之间新建锁定对齐……Creation.Document.NewGroup() 用于选择图元成组……LinearArray 和 RadialArray 两个类用于阵列图元。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第2章 2.5 节编辑图元（约 p108–p113）

---

## I — 方法论骨架 (Interpretation)

Revit 的"编辑图元"不是通用画布操作，而是一张"变换类型 → 专用 API 类"的映射表。旋转与镜像是 ElementTransformUtils 的静态方法（RotateElement / MirrorElements）；对齐是 ItemFactoryBase.NewAlignment()，在两个参照之间建立"锁定对齐"关系；成组是 Creation.Document.NewGroup()；阵列分 LinearArray 与 RadialArray 两类，各自又有"关联 Create / 独立 WithoutAssociation"两条路径；删除是 Document.Delete（返回实际被删 ElementId 集合）；固定是图元 Pinned 属性。

框架还内嵌一套"能力约束"：Pinned 图元不能旋转；组成员的 Move()/Rotate() 返回 true 但图元实际不动；镜像前必须 CanMirror 前置校验。核心心法：先问"这是什么变换"，再查"哪个类承认它"，最后做前置校验——而不是凭 UI 直觉直接调方法。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 旋转门/窗等族实例（2.5.1）
- **问题**: 需要把门绕轴旋转一个任意角度。
- **方法论的使用**: 识别为"旋转变换"→ 用 ElementTransformUtils.RotateElement，而不是实例自身提供的 Rotate()。
- **结论**: 旋转能力挂在工具类上，族实例自身不提供任意角度旋转。
- **结果**: 同一结论在第 3 章族实例旋转（revit-instance-rotate-location-type）再次验证。

### 案例 2: 弧线阵列一排构件（2.5.5）
- **问题**: 沿弧线放置一排构件并希望成员间保持参数关联。
- **方法论的使用**: 识别为"阵列"→ 选 RadialArray 的 Create（关联）而非 WithoutAssociation（独立）。
- **结论**: 关联成员随源图元更新，独立阵列生成后互不关联。
- **结果**: 选错构造器会得到"看起来一样但改一个不会联动"的阵列。

### 案例 3: 删除门后的级联报告（2.5.8）
- **问题**: 删除一个门，想知道哪些图元被连带删除。
- **方法论的使用**: Document.Delete 返回实际被删除的 ElementId 集合，直接读返回值。
- **结论**: 级联报告不必自行推断依赖关系。
- **结果**: 但删除的级联边界依关系类型而异，与 revit-foundation-host-deletion（删除楼板基础不删）形成对照。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要对构件做旋转/镜像/阵列/对齐/成组/删除/固定，但说不清该调哪个 API 类。
2. 旋转或移动"没反应"，需要排查 Pinned、组成员、CanMirror 等前置约束。
3. 删除构件后想输出"连带删了哪些"的级联清单。
4. 需要区分关联阵列（成员随源更新）与独立阵列。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么把门旋转 30 度" / "rotate this door 30 degrees"
- "沿弧线阵列一排柱" / "radial array these columns"
- "镜像这堵墙" / "mirror this wall"
- "为什么旋转没反应 / 调了 Move() 却不动" / "element doesn't move although I called Move()"
- "删除后怎么知道哪些被一起删了" / "what got deleted cascading"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **识别变换类型并查映射表**
   - 完成标准: 把需求对号入座为 旋转/镜像/对齐/成组/阵列/删除/固定 之一，写出对应 API 类与方法名。

2. **执行前置校验**
   - 完成标准: 旋转前确认图元非 Pinned、非组成员；镜像前调 CanMirror；删除前明确依赖关系。校验不通过则显式处理（解锁/移除组/跳过并记录）。
   - 判停条件: 若目标图元是组成员且无法移出组，跳到步骤 3 直接向用户报告原因，不再继续尝试。

3. **调用 API 并验证结果**
   - 完成标准: 旋转/镜像后读变换结果坐标或 Transformation；阵列后读新建实例数量；删除后读返回的 ElementId 集合；确认图元确实按预期改变、无异常抛出。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需平移或复制构件：那是移动/复制决策树的范畴（同一工具类的 Move/Copy 分支）。
- 需求是调整族实例的翻转/镜像属性（IsWorkPlaneFlipped），而非几何变换。
- 交互式手动编辑选中图元——本 skill 面向程序化变换。

### 作者在书中警告的失败模式

- Pinned 图元调用旋转会失败；必须先检查/取消固定。
- 组成员的 Move()/Rotate() 返回 true 但图元实际不动，是"静默不生效"陷阱。
- 镜像墙前若未做 CanMirror 校验，遇到无法镜像的图元会直接报错。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 / .NET 4.0：ElementTransformUtils 族的重载形态（如 RotateElement 的 origin/axis 参数）在后续版本中可能有调整；NewAlignment 在部分版本中标记废弃，落地前应核对当前 API 文档。

### 容易混淆的邻近方法论

- 对齐（Alignment）与移动（Move）：NewAlignment 创建"锁定对齐"关系，不是把图元挪过去；用途完全不同。
- 阵列的 Create 与 WithoutAssociation：都生成多个实例，只有 Create 保持参数关联。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
