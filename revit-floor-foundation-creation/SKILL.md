---
name: revit-floor-foundation-creation
description: |
  创建楼板/基础板/板报错或不知入口时。判别根是 FloorType.IsFoundationSlab：True → NewFoundationSlab；False → NewFloor/NewSlab。"类别相同不代表类相同"——楼板/基础板是系统族，单独基础是 FamilyInstance（NewFamilyInstance），墙基础是 ContFooting（支持有限），须按对象类选入口。不适用于：创建墙、单独基础构件族。
  Trigger："NewFoundationSlab 失败"、"怎么创建楼板/基础"、"同类别入口不同"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.2 楼板、天花板和基础（p147）
tags: [floor, foundation, type-discrimination, isfoundationslab, creation]
related_skills: []
---

# 楼板/基础板/板创建类型判别流程

## R — 原文 (Reading)

> 通过检索 FloorType 来创建楼板或基础板时，请使用图 3-3 中的方法。IsFoundationSlab 为 True → NewFoundationSlab；False → Floor/Slab 分支。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.2 节楼板、天花板和基础（p147）

---

## I — 方法论骨架 (Interpretation)

"板"在 Revit 里是一棵按判别属性分支的决策树，树的根是一个不起眼的布尔属性：FloorType.IsFoundationSlab。

True → 基础板，创建入口是 NewFoundationSlab；False → 普通楼板/天花板，走 NewFloor/NewSlab 分支。这个布尔属性是整个分支树的开关——先读它，再选创建方法，顺序反了就会调用失败。

更深的陷阱在于"类别≠类"：BuiltInCategory 相同（都叫"结构基础"）不代表对象类相同。楼板/基础板是系统族，用 FloorType 体系创建；独立基础是 FamilyInstance，要 NewFamilyInstance；墙基础是 ContFooting，API 支持有限。判别必须先分清"对象是什么类"，再谈"走哪个创建入口"。这个"判别属性→分支 API"的模式还在 Opening.IsRectBoundary、Grid.IsCurved 等图元上反复出现，是一族方法。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 基础板创建（3.1.2 图 3-3）
- **问题**: 需要创建基础板却不知道入口。
- **方法论的使用**: 检索 FloorType，读 IsFoundationSlab=true → NewFoundationSlab。
- **结论**: 创建入口由类型属性判别，不能凭名字猜。
- **结果**: 判别流程以流程图形式（图 3-3）给出。

### 案例 2: 单独基础 vs 系统族基础板（3.1.2）
- **问题**: 类别同为结构基础，为什么创建入口不同。
- **方法论的使用**: 识别单独基础是 FamilyInstance，走 NewFamilyInstance；墙基础是 ContFooting。
- **结论**: 按对象类选入口，不按 BuiltInCategory。
- **结果**: 同类别的图元可能是完全不同的类。

### 案例 3: 判别属性模式跨图元复现
- **问题**: 如何判断相似 API 的用法。
- **方法论的使用**: IsFoundationSlab / IsRectBoundary / IsCurved 同为"判别属性→分支 API"模式。
- **结论**: 掌握模式比记单个属性更可靠。
- **结果**: 见 revit-opening-boundary-reading、revit-grid-curve-creation。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 创建楼板/天花板/基础板，不确定用哪个 API。
2. NewFoundationSlab 调用失败，怀疑选错了入口。
3. 看到"结构基础"类别的图元，要判断该按哪条路径处理。
4. 写"按类别批量创建"的通用工具。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么创建基础板" / "create a foundation slab"
- "NewFoundationSlab 为什么会失败" / "why does NewFoundationSlab fail"
- "楼板和基础板创建有什么区别" / "floor vs foundation slab creation"
- "IsFoundationSlab" / "判别属性"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **先判对象类**
   - 完成标准: 确认要创建的是 FloorType 体系（楼板/基础板）、FamilyInstance（单独基础）还是 ContFooting（墙基础）；不要按 BuiltInCategory 下结论。
   - 判停条件: 若目标是单独基础，跳到 NewFamilyInstance 流程；若是墙基础，确认 API 支持范围后再动手。

2. **读判别属性**
   - 完成标准: 对 FloorType 读 IsFoundationSlab，true 走 NewFoundationSlab，false 走 NewFloor/NewSlab。

3. **创建并验证**
   - 完成标准: 创建成功返回非 null；用 IsFoundationSlab 验证结果类型与预期一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 创建普通墙/幕墙：那是墙的创建流程。
- 创建独立基础构件族：那是 NewFamilyInstance 的范畴，不在本判别树内。

### 作者在书中警告的失败模式

- NewFoundationSlab 只接受 IsFoundationSlab=true 的 FloorType——传普通楼板类型会失败。
- 按 BuiltInCategory 猜类：同类别下藏着多个类与多条创建路径，猜即错。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：ContFooting 的支持有限，在新版本中 API 能力有所增强但仍是特殊路径；独立基础在新版中也有更丰富的创建参数，落地以当前 API 为准。

### 容易混淆的邻近方法论

- "基础"一词的三义：基础板（FloorType 系统族）、独立基础（FamilyInstance）、墙基础（ContFooting）——同一个词三个类三条路径。
- 类型判别 vs 实例判别：IsFoundationSlab 挂在 FloorType（类型级），不是实例属性。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
