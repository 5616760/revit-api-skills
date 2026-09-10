---
name: revit-link-reference-conversion
description: |
  链接文件拾取图元/面后要在主体文件基于它工作时：用 Selection.PickObject(ObjectType.LinkedElement) 取链接内 Reference，再 CreateLinkReference() 转为主体 Reference。信号："链接里选面 / pick element in link / linked geometry reference"、"跨链接参照 / CreateLinkReference"。不适用：修改链接内图元、纯主体内拾取。
  Trigger: CreateLinkReference / LinkedElementId / 链接里选面。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.15.1 链接几何图形（约p413-414）
tags: [revit-api, reference, linked-element, selection-filter, geometry]
related_skills: []
---

# 链接图元交互（Reference 转换）决策

## R — 原文 (Reading)

> 转换几何参照（Conversion of Geometric References）。Reference 类拥有相关链接文件的成员，允许在仅参照链接内容的 Reference 对象和参照主体的 Reference 对象之间进行转换。
>
> — 宦国胜, 第5章 5.15.1 链接几何图形（约p413-414）

---

## I — 方法论骨架 (Interpretation)

链接文件里的几何"看得见、选得着，但不能直接用"——因为拾取到的 Reference 属于链接文档的坐标系与图元空间，而 `NewFamilyInstance` 等主体 API 只认主体文档的 Reference。所以跨链接交互的核心动作是**Reference 转换**，配套三件工具：

1. **拾取**：`Selection.PickObject(ObjectType.LinkedElement, ISelectionFilter)` 让用户直接在链接里点图元，返回链接侧 Reference；ISelectionFilter 的 `AllowElement`/`AllowReference` 两层回调可限定只允许选面/墙等。
2. **转换**：`Reference.CreateLinkReference()` 把链接侧 Reference 转为主体文档的对应 Reference；反向则用 `LinkedElementId` 从主体侧回到链接图元。典型用法：在链接里找到目标面 → 转换为主体参照 → 用它放置基于面的族实例。
3. **保存策略**：保存共享坐标相关的链接交互可挂 `ISaveSharedCoordinatesCallback`，由回调决定何时允许保存。

红线：`GetLinkDocument()` 返回的链接文档是**只读**的——想改链接内容必须打开那个链接文件本身，而不是在主体文档里改。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 在链接中点选一个面并在主体中放置基于面的族
- **问题**: 用户想在链接模型的某个面上放族实例（如设备），但拾取的 Reference 属于链接。
- **方法论的使用**: `PickObject(ObjectType.LinkedElement, filter)` 取链接内 Reference → `Reference.CreateLinkReference()` 转换为主体 Reference → 将主体 Reference 作为 `NewFamilyInstance` 的参照输入。
- **结论**: 链接几何可以被"借用作参照"，但不能被修改。
- **结果**: 族实例成功落在由链接面推得的主体现场上，实现基于链接的半自动放置。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件让用户在链接模型中点选面/墙，并据此在主体文件放置构件或标注。
2. 需要根据链接内图元做测量、碰撞或对齐，需要把拾取结果接入主体文档的几何 API。
3. 保存共享坐标时机不确定，需要自定义"何时允许保存"的交互策略。
4. 用户抱怨"选了链接里的东西，代码却放不了族/取不到几何"。

### 语言信号 (用户的话里出现这些就应激活)

- "在链接里选/拾取" / "pick element in linked file / select in link"
- "链接的面转成参照" / "convert link reference / CreateLinkReference"
- "用链接的面放族" / "place family instance on linked face"
- "链接文档只读、改不了" / "link document is read-only"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **限定拾取范围**
   - 实现 ISelectionFilter（AllowElement/AllowReference），调用 `Selection.PickObject(ObjectType.LinkedElement, filter)` 让用户在链接中拾取。
   - 完成标准: 拿到一个指向链接内图元的 Reference。
   - 判停条件: 若用户要"修改链接里的图元"，停下说明链接文档只读，需打开链接文件本身操作。

2. **转换为文档 Reference**
   - 调用 `Reference.CreateLinkReference()` 得到主体文档的对应 Reference（需要链接实例上下文）。
   - 完成标准: 转换后的 Reference 能传入主体文档 API（如 NewFamilyInstance）且不抛异常。

3. **用主文档 Reference 执行后续操作**
   - 把转换后的 Reference 作为放置/测量/对齐的输入执行业务逻辑；涉及共享坐标保存时实现 ISaveSharedCoordinatesCallback 控制保存时机。
   - 完成标准: 目标构件/标注成功落在主体现场，或几何计算返回有效结果。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标是修改链接文件内的图元——`GetLinkDocument()` 只读，必须打开链接文档本身（在其中有写权限的会话）。
- 纯主体内拾取（无链接参与）——直接用普通 `PickObject(ObjectType.Element)` 即可，无需转换。
- 剖切边缘等本身不提供 Reference 的对象——先做有效性判断，别硬转换。

### 作者在书中警告的失败模式

- 拿链接侧 Reference 直接喂给主体 API（如 NewFamilyInstance），坐标系/文档不匹配导致放置失败或异常。
- 以为能通过链接文档修改链接内容——只读约束被忽略，写入抛异常。
- ISelectionFilter 只实现了一层过滤（AllowElement），没过滤 AllowReference，用户选到不可用参照。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：新版对链接拾取与 Worksharing/云模型下链接参照的行为有增强（如 LinkedElementId 语义、重新加载链接后 Reference 稳定性），需要核对当前版本。
- 书中对 ISaveSharedCoordinatesCallback 的适用时机着墨有限，实践细节需自行验证。

### 容易混淆的邻近方法论

- `revit-link-decision-flow`（链接加载与创建决策）——那是文件层，本 skill 是几何参照层。
- `revit-transmissiondata-offline`（离线链接状态）——与本 skill 无交集，避免混用场景。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
