---
name: revit-foundation-host-deletion
description: |
  批量删除楼板后模型里出现"孤立基础"时。宿主-基础关系不在删除级联范围内：删除主体楼板时，基础并不随之删除。基础主体要从 Host（INSTANCE_FREE_HOST_PARAM）参数获得——不是常见的 Host 属性。清理工具不能只删楼板，必须显式遍历基础图元、读 INSTANCE_FREE_HOST_PARAM 判断是否已孤立再决定删除；依赖自动级联会留垃圾数据。
  不适用于：一般删除级联报告（那是 Document.Delete 返回值）。
  Trigger："删了楼板还有孤立基础"、"基础不跟着楼板删"、"INSTANCE_FREE_HOST_PARAM"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.2 楼板、天花板和基础（p148）
tags: [deletion, foundation, host-relationship, cascading-delete, cleanup]
related_skills:
  - slug: revit-element-transform-utils
    relation: contrasts-with
---

# 删除主体楼板时基础不随之删除

## R — 原文 (Reading)

> 当删除主体楼板时，基础并不随它一起删除。基础主体可以从 Host（INSTANCE_FREE_HOST_PARAM）参数获得。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.2 节楼板、天花板和基础（p148）

---

## I — 方法论骨架 (Interpretation)

"删了宿主，附着的图元也会删"是一个危险的直觉——它只在部分关系上成立。Revit 的删除级联边界按关系类型而定：有些关系会级联（如墙洞、嵌套的子图元），但宿主-基础关系不在其中。删除主体楼板，基础会原地留下，变成"孤立基础"。

查基础归属于谁，用的是 INSTANCE_FREE_HOST_PARAM 参数（所谓 Host 参数的"自由实例"形态），而不是通用的 Host 属性——这是与 UI 直觉相反的 API 约定。

因此清理工具的正确写法是：不能假设删除会级联，必须显式遍历所有基础图元，读 INSTANCE_FREE_HOST_PARAM 判断当前宿主是否还存在（被删了、为 null 就是孤立），再决定是否删除。依赖任何自动级联，都会在模型里留下垃圾数据。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 孤立基础的成因（3.1.2）
- **问题**: 批量删楼板后模型里多出一堆孤立基础。
- **方法论的使用**: 确认宿主-基础关系不在删除级联范围内。
- **结论**: 级联边界依关系类型而异，不能一概而论。
- **结果**: 删宿主≠删全部关联。

### 案例 2: 查询基础主体（3.1.2）
- **问题**: 找到基础归属于哪个楼板。
- **方法论的使用**: 读 INSTANCE_FREE_HOST_PARAM 参数。
- **结论**: 用参数而非 Host 属性查宿主关系。
- **结果**: 与 UI 直觉相反，但 API 如此。

### 案例 3: 与文档级删除级联的对照（2.5.8）
- **问题**: 哪些删除会级联？
- **方法论的使用**: Document.Delete 会级联删除依赖子图元（s2-x02），但基础不属于这类。
- **结论**: 两种关系、两种边界，形成对照。
- **结果**: 共同支撑"级联边界依关系类型而定"的方法论。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写批量删除楼板/结构件的清理工具。
2. 删除后发现模型残留孤立基础/构件。
3. 查询基础构件归属于哪个宿主。
4. 判断一段删除操作会不会连带删掉关联图元。

### 语言信号 (用户的话里出现这些就应激活)

- "删了楼板还有孤立基础" / "orphan foundations left after deleting slabs"
- "基础不跟着楼板删" / "foundations are not deleted with the host slab"
- "怎么查基础的宿主" / "get the host of a foundation"
- "INSTANCE_FREE_HOST_PARAM" / "删除级联边界"

### 与相邻 skill 的区分

- 与 `revit-element-transform-utils`：本 skill 讲删除时哪些宿主/子构件关系不级联及清理写法，后者讲 Document.Delete 级联返回值与变换工具——删除语义相对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **不依赖级联**
   - 完成标准: 确认当前删除操作不会自动带走基础；需要清理时按步骤 2 显式处理。

2. **遍历并判孤立**
   - 完成标准: 遍历基础图元，读 INSTANCE_FREE_HOST_PARAM 得宿主 ElementId；宿主已不存在（null/被删）即判定孤立。

3. **清理并验证**
   - 完成标准: 只删除判定为孤立的图元；删除后模型无残留孤立基础；未误删仍有宿主的基础。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 一般删除操作的级联报告：走 Document.Delete 返回值。
- 创建/判别基础类型：那是 revit-floor-foundation-creation。

### 作者在书中警告的失败模式

- 假设"删宿主=删全部关联"：宿主-基础关系不在级联范围，留下孤立基础。
- 用 Host 属性查基础宿主：必须用 INSTANCE_FREE_HOST_PARAM，用错查不到。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：INSTANCE_FREE_HOST_PARAM 在后续版本仍是查询宿主关系的标准路径；孤立清理的语义保持稳定。

### 容易混淆的邻近方法论

- "级联删除"与"不级联删除"：两种关系两种边界，判断依据是关系类型而非图元类别。
- Host 属性 vs INSTANCE_FREE_HOST_PARAM：前者是通用宿主属性，后者是基础-楼板宿主关系的参数化表达，不要互换。

---

## 相关 skills

- **revit-element-transform-utils**（contrasts-with）：本 skill 讲删除时哪些宿主/子构件关系不级联及清理写法，后者讲 Document.Delete 级联返回值与变换工具——删除语义相对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
