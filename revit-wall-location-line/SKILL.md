---
name: revit-wall-location-line
description: |
  创建/修改墙体并关心定位线时。WALL_KEY_REF_PARAM 是整型 0–5（0 墙中心线/1 核心中心线/2 端面外/3 端面内/4 核心面外/5 核心面内，非枚举）；flip() 翻转 Orientation。改定位线动位置不动几何关系；位移量用 CompoundStructure.GetOffsetForLocationLine() 算出。不适用于：改墙体类型结构、墙几何提取。
  Trigger："墙定位线"、"WALL_KEY_REF_PARAM"、"改定位线移动多少"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.1 墙（p145–p146）
tags: [wall, location-line, wallkeyrefparam, orientation, wall-creation]
related_skills:
  - slug: revit-compound-structure-layers
    relation: depends-on
---

# 墙定位线 WALL_KEY_REF_PARAM 取值与几何控制

## R — 原文 (Reading)

> 墙定位线（WALL_KEY_REF_PARAM）参数是3……定位线的值：0 墙中心线；1 核心中心线；2 端面：外部；3 端面：内部；4 核心面：外部；5 核心面：内部。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.1 节墙（p145–p146）

---

## I — 方法论骨架 (Interpretation)

墙体不是简单的一根粗线，它有一条"定位线"（Location Line）作为与标高的锚定协议：墙体厚度方向的哪个位置"坐"在参考点上。这个位置是 WALL_KEY_REF_PARAM 参数，取值 0–5 六个档位（墙中心线、核心中心线、端面外/内部、核心面外/内部）。两个要点：

其一，它是整型编码不是枚举——UI 里选"定位线"就是写这个整型参数，API 里同样用整数访问，别幻想有 LocationLine 枚举可转。其二，改定位线改变的是墙体的"位置"（墙体整体平移，让选定线落到参考点上），墙体相对自身定位线的几何关系不变；移动了多少不必试错，CompoundStructure.GetOffsetForLocationLine() 能从中心定位线直接算出到任一档位的偏移量。另有一条方向约定：Wall.flip() 把 Orientation 从 (0,1,0) 翻成 (0,-1,0)。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建带定位线的墙（3.1.1）
- **问题**: 创建墙时希望按"核心面外部"定位。
- **方法论的使用**: 设定 WALL_KEY_REF_PARAM 整型值（4），按锚定协议放置墙体。
- **结论**: 定位线是墙与标高的锚定协议，取值 0–5 整型编码。
- **结果**: 定位线语义在复合结构语境中由 GetOffsetForLocationLine 独立复现。

### 案例 2: 计算定位线位移（3.1.6）
- **问题**: 把定位线从中心线改为核心面外部，预算墙体位移。
- **方法论的使用**: CompoundStructure.GetOffsetForLocationLine() 从复合结构直接算出偏移。
- **结论**: 位移量是复合结构层的几何属性，可精确预算。
- **结果**: 见 revit-compound-structure-layers 的配套验证。

### 案例 3: 翻转方向（3.1.1）
- **问题**: 需要反转墙的方向。
- **方法论的使用**: 调 flip()，Orientation 从 (0,1,0) 变 (0,-1,0)。
- **结论**: 翻转是定位线语义的一部分。
- **结果**: 翻转后的墙体保持定位线锚定关系。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 创建墙时明确要按某种定位线放置（如"核心面外部对齐"）。
2. 读取/修改已有墙的定位线取值，并预判改后的位移。
3. 墙体翻转方向的程序化处理。
4. 算量/净空检查需要知道墙身相对定位线的位置。

### 语言信号 (用户的话里出现这些就应激活)

- "墙的定位线怎么设" / "set the wall location line"
- "WALL_KEY_REF_PARAM 是什么" / "what is the wall key ref param"
- "把定位线改成核心面外部会移动多少" / "how much will the wall move if I change the location line"
- "flip the wall orientation"

### 与相邻 skill 的区分

- 与 `revit-compound-structure-layers`：本 skill 的定位线位移计算依赖其 GetOffsetForLocationLine 层偏移接口。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **读/写定位线参数**
   - 完成标准: 用 WALL_KEY_REF_PARAM 整型 0–5 读写；能说出每个取值的含义；确认不是枚举类型。

2. **预判位移或翻转**
   - 完成标准: 需要改定位线时，用 CompoundStructure.GetOffsetForLocationLine() 计算目标档位相对中心线的偏移；需要反转时调 flip() 并验证 Orientation 翻转。

3. **验证墙体位置**
   - 完成标准: 修改后墙体位置与预期偏移一致；定位线与参考点的锚定关系正确；无异常。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 修改墙体的层结构/材料：那是 revit-compound-structure-layers 的职责。
- 创建墙本身而无需控制定位线：走 revit-wall-create-overloads 的默认即可。

### 作者在书中警告的失败模式

- 把定位线当成枚举处理（如直接转 LocationLine）：它是整型参数，编码映射错位会静默选错档位。
- 假设"改定位线=改几何"：改的是位置，墙体相对定位线的几何关系不变。
- 用试错法测位移：有现成 API 可精确计算，试错既慢又易残留错误墙位。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：WALL_KEY_REF_PARAM 的 0–5 编码在后续版本保持；但新版本中复合结构的层功能（Function）枚举有扩展，GetOffsetForLocationLine 的输入校验以当前 API 为准。

### 容易混淆的邻近方法论

- 定位线（Location Line）与驱动线（LocationCurve）：前者是墙厚度方向的位置协议，后者是墙几何的轴线，同名相近但作用不同。
- flip() 翻转方向与镜像变换：flip 只翻 Orientation，不产生镜像副本。

---

## 相关 skills

- **revit-compound-structure-layers**（depends-on）：本 skill 的定位线位移计算依赖其 GetOffsetForLocationLine 层偏移接口。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
