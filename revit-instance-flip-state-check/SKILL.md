---
name: revit-instance-flip-state-check
description: |
  用户要翻转族实例朝向/把手/工作平面（如批量把门开向翻转却部分失败）、设置 flip 相关属性报异常、或想用翻转复现镜像效果时调用。不适用于：整段几何镜像（用 ElementTransformUtils.MirrorElements）、非族实例的旋转移动。关键 trigger："翻转门/窗的开向"、"flip the door"、"CanFlipFacing"、"CanFlipHand"、"为什么部分实例翻转失败"、"翻转和镜像什么关系"。核心规则：先查 Can* 能力位再调 flip 方法，对不允许翻转的实例设属性会抛异常。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.3 族实例（p160）
tags: [familyinstance, flip, facing, hand, workplane, revit-api]
related_skills:
  - slug: revit-instance-rotate-location-type
    relation: contrasts-with
  - slug: revit-element-transform-utils
    relation: contrasts-with
---

# 族实例朝向/把手/工作平面翻转状态检查流程

## R — 原文 (Reading)

> 如果 CanFlipFacing 或 CanFlipHand 为 true，则 flipFacing() 或 flipHand() 方法可分别调用……如果对不允许工作平面翻转的族实例设置此属性，则会引发异常。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.3 族实例（p160）

---

## I — 方法论骨架 (Interpretation)

族实例有**三个可翻转的轴**，每个轴都配一个"能力位检查 + 翻转动作"：

| 轴 | 能力位检查 | 翻转动作 |
|---|---|---|
| 朝向 Facing | CanFlipFacing | flipFacing() |
| 把手 Hand | CanFlipHand | flipHand() |
| 工作平面 WorkPlane | CanFlipWorkPlane | 设 IsWorkPlaneFlipped |

铁律：**先查 Can\*，为 true 才动作**。这不是可选优化——对不支持翻转的实例强行设属性会直接抛异常。

由此推出两个实战推论：
1. **门有 4 种朝向状态**：FacingFlipped × HandFlipped 两轴各 2 态的组合，不是简单的"正/反"。
2. **翻转 = 镜像变换**：镜像后的门可以用翻转把手方向复现，不必删除重建。这也解释了"批量翻转部分失败"——家具等族可能本身不支持翻转。

这个 Can\* 前置校验模式在全书反复出现（CanRotate、CanMirror、CanAddViewToSheet），是 Revit API 的统一习惯。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量翻转门开向
- **问题**: 想批量翻转项目里所有门的开向，但部分失败。
- **方法论的使用**: 逐实例先查 CanFlipHand / CanFlipFacing，仅对 true 的调用 flip。
- **结论**: 失败源于未做能力位检查——不是所有族都支持翻转。
- **结果**: 加前置校验后批量翻转稳定完成。

### 案例 2: 用翻转复现镜像
- **问题**: 一扇镜像过的门，希望得到等效朝向。
- **方法论的使用**: 镜像等价于翻转把手方向，走 flipHand 而非删除重建。
- **结论**: 翻转即镜像变换，两轴组合可覆盖 4 种朝向。
- **结果**: 用翻转复现镜像态，几何与标签方向都正确。

### 案例 3: 工作平面翻转的异常
- **问题**: 对某些实例设置工作平面翻转属性报异常。
- **方法论的使用**: 检查 CanFlipWorkPlane，不允许则跳过。
- **结论**: 属性设置不是总能成功，异常即最明确信号。
- **结果**: 用 Can\* 位把非法操作挡在异常之前。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批量翻转门/窗开向，部分实例失败或想预判失败。
2. 写代码设 flipFacing / flipHand / IsWorkPlaneFlipped 前需要安全校验。
3. 想搞清楚门/家具的朝向状态有几种。
4. 镜像构件后想用翻转等效复现，避免重建。

### 语言信号 (用户的话里出现这些就应激活)

- "翻转门的开向" / "flip the door swing"
- "为什么有的门翻不过来" / "why can't I flip this instance"
- "CanFlipFacing / CanFlipHand / CanFlipWorkPlane"
- "翻转和镜像" / "flip vs mirror"
- "设置 flip 属性抛异常"

### 与相邻 skill 的区分

- 与 `revit-instance-rotate-location-type`：本 skill 管三轴翻转的二元状态，后者管角度旋转与位置读写，能力正交、互为替代。
- 与 `revit-element-transform-utils`：翻转是实例属性操作而非几何变换 API，后者提供 Move/Copy/Rotate 等工具，实现手段对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确翻转轴**
   - 目标轴是 Facing / Hand / WorkPlane 中的哪个，对应读哪个 Can\*。
   - 完成标准: 写出目标轴与对应能力位、对应方法/属性的映射。

2. **先查能力位，再动作**
   - `if (inst.CanFlipFacing) inst.flipFacing();` 逐轴处理。
   - 需要镜像效果 → flipHand；需要 4 态遍历 → 两轴组合。
   - 完成标准: 每个翻转调用前都有对应 Can\* 守卫。

3. **异常与结果校验**
   - 捕获/记录跳过的不支持实例；事务内完成后抽查实例朝向已变。
   - 完成标准: 目标实例全部翻转或明确记录跳过原因。
   - 判停条件: 若需要整段几何镜像而非姿态翻转，则改用 ElementTransformUtils.MirrorElements。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 要镜像整个构件/几何体，翻转是姿态操作、镜像是几何变换，两者目标不同。
- 非族实例（墙、轴网）没有 Facing/Hand 概念。

### 作者在书中警告的失败模式

- 对不支持翻转的实例设置属性会**引发异常**——Can\* 是硬性前置，不是可选优化。
- 翻转不等价于所有情况的镜像（文本/嵌套子构件方向可能不同）。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：三轴翻转模型延续，但部分方法签名/新实例类型（如 MEP 连接件翻转）在后续版本扩展。

### 容易混淆的邻近方法论

- 旋转（Rotate）与翻转（Flip）是不同能力：旋转改变角度，翻转只切换二元朝向。
- 翻转结果可用镜像复现，但不要反过来认为镜像 API 是翻转的替代。

---

## 相关 skills

- **revit-instance-rotate-location-type**（contrasts-with）：本 skill 管三轴翻转的二元状态，后者管角度旋转与位置读写，能力正交、互为替代。
- **revit-element-transform-utils**（contrasts-with）：翻转是实例属性操作而非几何变换 API，后者提供 Move/Copy/Rotate 等工具，实现手段对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
