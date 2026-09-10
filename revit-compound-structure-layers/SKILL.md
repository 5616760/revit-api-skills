---
name: revit-compound-structure-layers
description: |
  读取/修改墙/楼板/屋顶类型的层结构时：从 HostObjAttributes（WallType 等）取 CompoundStructure，GetLayers 读层、SetCompoundStructure 写回。类型级=全局级，改一层全楼同类型都变，要局部影响先 ElementType.Duplicate() 复制再改。MaterialId=-1 是"材料与类别相关"，非错误值。不适用于：改实例外观、热属性。
  Trigger："改层宽全楼都变"、"MaterialId=-1"、"读复合结构层"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.6 复合结构（p152–p153）
tags: [compound-structure, hostobjattributes, wall-layers, elementtype, duplicate]
related_skills: []
---

# 复合结构 CompoundStructure 层读取与修改流程

## R — 原文 (Reading)

> 使用复合结构，可找到各层边界的几何位置。GetOffsetForLocationLine() 提供从中心定位线到任何定位线选项的偏移。须将复合结构返回到 HostObjAttributes……合并新层须用 ElementType.Duplicate() 新建 HostObjAttributes。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.6 节复合结构（p152–p153）

---

## I — 方法论骨架 (Interpretation)

复合结构（CompoundStructure）是墙/楼板/屋顶这些"多层构造"的类型级几何骨架：它挂在 HostObjAttributes（WallType、FloorType 等）上，定义每一层的材料、厚度、功能，也定义了各层边界的几何位置。

访问流程固定三条：读——从类型对象拿 CompoundStructure，GetLayers() 读出层数组；改——修改层对象后 SetCompoundStructure() 写回类型；隔离——如果不想影响全楼，先 ElementType.Duplicate() 复制出一个新 HostObjAttributes，再在副本上改，然后赋给目标实例。

两条语义必须记住：其一，类型级 = 全局级，改类型上的任何层，所有用该类型的实例都变——"为什么改一层全楼都变"是正常的，隔离靠复制类型；其二，MaterialId = -1 不是错误，它表示"材料与类别相关"，按层的 Function 取类别的默认材料。另外 GetOffsetForLocationLine() 把复合结构的层边界几何与墙定位线联动起来，位移可精确预算。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 读层边界几何（3.1.6）
- **问题**: 需要知道各层边界的几何位置。
- **方法论的使用**: 从 HostObjAttributes 取 CompoundStructure，用 GetOffsetForLocationLine 算边界偏移。
- **结论**: 复合结构是层边界几何的来源。
- **结果**: 与 revit-wall-location-line 的定位线位移计算联动。

### 案例 2: 新建层需要复制类型（3.1.6）
- **问题**: 要一个"带新合并层"的墙类型而不影响现有墙。
- **方法论的使用**: ElementType.Duplicate() 新建 HostObjAttributes 再改。
- **结论**: 类型级修改必须靠复制隔离。
- **结果**: 避免全楼墙体的连锁变化。

### 案例 3: 类型级属性族（3.1.8 热属性）
- **问题**: 读墙体热工属性。
- **方法论的使用**: ThermalProperties 同样挂在 WallType/FloorType/CeilingType/RoofType 上。
- **结论**: 类型级属性是 Revit 的通用访问模式。
- **结果**: 见 s2b-t04。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 改一个墙类型的层宽，结果全楼墙体都变了，需要解释并隔离。
2. 读取墙/楼板类型的层结构（材料、厚度、功能）做分析。
3. 读到 MaterialId = -1，怀疑数据错误。
4. 计算墙定位线改档位后的层边界位移。

### 语言信号 (用户的话里出现这些就应激活)

- "改墙类型的层宽全楼都变了" / "changing a wall type layer affects every wall"
- "怎么读复合结构的层" / "read compound structure layers"
- "MaterialId 是 -1" / "material id is -1"
- "新建一个墙类型再改层" / "duplicate a wall type before modifying layers"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **决定是否隔离**
   - 完成标准: 若修改会波及全楼且不希望如此，先 ElementType.Duplicate() 复制新 HostObjAttributes；确认复制后类型已独立。

2. **读/改层**
   - 完成标准: 取 CompoundStructure；GetLayers() 读出层；修改层对象（材料/厚度/功能）后 SetCompoundStructure() 写回；MaterialId=-1 时按 Function 解析类别默认材料而非报错。

3. **验证影响范围**
   - 完成标准: 确认只改了目标类型/实例；原类型不受影响；层边界几何偏移（如需）用 GetOffsetForLocationLine 校验。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 修改单个实例的外观材质（非结构层）：那是实例参数或外观参数范畴。
- 计算热工性能：热属性是另一套 API（s2b-t04）。

### 作者在书中警告的失败模式

- 改类型层＝全局生效，不复制类型就动手会波及所有实例。
- MaterialId=-1 被误判为错误值——它是"类别相关材料"的编码。
- 垂直复合墙（IsVerticallyCompound）需要不同的层处理路径，按普通墙处理会漏层。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：IsVerticallyCompound 等垂直复合能力在新版本中更常见，相关校验逻辑以当前 API 文档为准；GetOffsetForLocationLine 签名保持稳定。

### 容易混淆的邻近方法论

- 层（Layer）与材质（Material）：层是构造单元，材质是层的内容之一；改层宽 vs 改材质是两个不同操作。
- Duplicate() 与直接改：复制是"造新类型"，直接改是"改旧类型"，隔离与否取决于意图。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
