---
name: revit-wall-create-overloads
description: |
  创建墙不确定用哪个 Wall.Create 重载时。决策输入是"手里参数形态"：只有标高 ElementId → 用带 ElementId 重载 Create(Document, Curve, ElementId, bool)，不必先 GetElement；矩形轮廓走曲线重载，非矩形走 IList<Curve> 重载，要法向用带 XYZ 重载（表 3-2 共 5 种）。决策模式与 NewOpening、NewFamilyInstance 同构。
  Trigger："怎么创建墙"、"create wall with only element id"、"非矩形轮廓墙"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.1 墙（p146）
tags: [wall, create-overloads, overload-selection, api-design, wall-creation]
related_skills:
  - slug: revit-floor-foundation-creation
    relation: contrasts-with
  - slug: revit-wall-location-line
    relation: composes-with
  - slug: revit-wall-structural-usage-rules
    relation: composes-with
---

# 墙 Create 重载方法选择决策

## R — 原文 (Reading)

> 在 Wall 类中有五种静态重载方法来创建墙，详见表 3-2。矩形轮廓墙、非矩形轮廓墙、指定标高、指定类型和法向向量等。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.1 节墙（p146）

---

## I — 方法论骨架 (Interpretation)

Wall.Create 不是"一个函数，五个参数可省略"，而是五个按"几何约束组合"切分的静态重载：轮廓是矩形还是任意闭合曲线、标高是默认（活动标高）还是显式指定、类型是默认类型还是指定类型、是否要显式法向向量——每个组合对应一个签名。

决策的关键心法：选哪个重载，取决于"你手上有什么"，而不是"你想要什么"。例如只有标高 ElementId 而没有 Level 对象时，选带 ElementId 参数的 Create(Document, Curve, ElementId, bool)，不必先 GetElement 换对象；要创建带法向的非矩形轮廓墙，则用第 5 重载 Create(Document, IList<Curve>, ElementId, ElementId, bool, XYZ)。

这个"按参数形态选重载"的决策模式不是墙独有的——NewOpening 按主体类型切 4 个重载、NewFamilyInstance 按放置特征切 12 个重载，同一设计哲学在 Revit API 里反复出现。遇到"哪个构造函数"的困惑时，先列出自己手上所有对象的类型，再对照重载表。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 矩形轮廓墙（3.1.1 表 3-2）
- **问题**: 创建最常见的直线/弧线矩形轮廓墙。
- **方法论的使用**: 用曲线（Curve）+ 标高对象/ElementId 的基础重载。
- **结论**: 基础重载覆盖 90% 的日常墙创建。
- **结果**: 默认类型、默认标高由系统补全。

### 案例 2: 非矩形轮廓 + 法向（3.1.1 表 3-2）
- **问题**: 创建异形轮廓墙并控制其朝向。
- **方法论的使用**: 第 5 重载 Create(Document, IList<Curve>, ElementId, ElementId, bool, XYZ)。
- **结论**: 非矩形轮廓必须走 IList<Curve> 入口。
- **结果**: 与矩形轮廓的重载分工明确。

### 案例 3: 只有 ElementId（V2 推演）
- **问题**: 手里只有标高 ElementId 而没有 Level 对象。
- **方法论的使用**: 选带 ElementId 参数的 Create(Document, Curve, ElementId, bool)，不先 GetElement。
- **结论**: 重载表直接消化"对象缺失"问题。
- **结果**: 决策输入是持有的参数类型。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写创建墙的代码，面对多个重载不知选哪个。
2. 手里只有 ElementId（标高/类型），没有对应的 Level/WallType 对象。
3. 需要创建非矩形轮廓墙（异形轮廓）。
4. 需要控制墙的法向向量。

### 语言信号 (用户的话里出现这些就应激活)

- "创建墙用哪个重载" / "which Wall.Create overload should I use"
- "只有标高的 ElementId 怎么建墙" / "create a wall with only a level element id"
- "非矩形轮廓的墙怎么建" / "create a wall with a non-rectangular profile"
- "wall create overloads" / "表3-2"

### 与相邻 skill 的区分

- 与 `revit-floor-foundation-creation`：本 skill 管墙的创建重载选择，后者管楼板/基础的类型判别创建——对象不同但决策框架同构。
- 与 `revit-wall-location-line`：创建墙时组合其定位线参数来确定墙的位置语义。
- 与 `revit-wall-structural-usage-rules`：墙创建后其结构用法/房间边界的推导规则与本 skill 组合成完整墙流程。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **清点手上参数**
   - 完成标准: 列出拥有的对象类型：Curve 或 IList<Curve>、Level 或 ElementId、WallType 或默认、是否要法向 XYZ。

2. **对照重载表选签名**
   - 完成标准: 按"轮廓形状 × 标高形态 × 类型来源 × 法向"组合选出唯一匹配的 Wall.Create 重载。
   - 判停条件: 若只有 ElementId 且不想换对象，确认所选重载含 ElementId 参数，不必先 GetElement。

3. **创建并验证**
   - 完成标准: 调用成功，返回非 null 的 Wall；墙的标高、类型、轮廓与预期一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 创建的不是墙（楼板/基础板）：那是 revit-floor-foundation-creation 的判别流程。
- 修改已有墙的几何/类型：不是创建范畴。

### 作者在书中警告的失败模式

- 想按"功能需求"找重载，结果在重载名之间迷失——决策输入是参数形态，不是语义。
- 非矩形轮廓硬塞进单曲线重载：编译或运行期失败，轮廓必须是 IList<Curve>。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：5 个重载的清单在后续版本基本保持，但个别签名（如 ElementId 与 Level 的混载）在 .NET 版本演进中有细微调整，落地前核对当前 IntelliSense。

### 容易混淆的邻近方法论

- 重载选择 vs 类型选择：重载是"用哪个签名"，类型是"用哪种 WallType"，两层不同，别混为一谈。
- 矩形/非矩形轮廓：区别在"一条 Curve"还是"一组闭合曲线"，不是视觉上的直/弧。

---

## 相关 skills

- **revit-floor-foundation-creation**（contrasts-with）：本 skill 管墙的创建重载选择，后者管楼板/基础的类型判别创建——对象不同但决策框架同构。
- **revit-wall-location-line**（composes-with）：创建墙时组合其定位线参数来确定墙的位置语义。
- **revit-wall-structural-usage-rules**（composes-with）：墙创建后其结构用法/房间边界的推导规则与本 skill 组合成完整墙流程。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
