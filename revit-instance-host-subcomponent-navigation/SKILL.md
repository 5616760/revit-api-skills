---
name: revit-instance-host-subcomponent-navigation
description: |
  用户要拿实例的主体墙/宿主表面、展开嵌套族找子构件、理解 Host 返回 null、或写递归遍历嵌套族工具时调用。不适用于：只要实例自身姿态/类型、只过滤无宿主图元而无需关系导航的场景。关键 trigger："门的宿主墙怎么拿"、"get the host of an instance"、"嵌套族里的子构件"、"GetSubComponentIds"、"HostFace 是什么"、"SuperComponent"、"为什么 Host 是 null"。核心：Host/HostFace 管宿主关系（无宿主返回 null）、Sub/SuperComponent 管嵌套关系，导航代码必须容忍 null。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.3 族实例（p162）
tags: [familyinstance, host, subcomponent, supercomponent, nested-family, revit-api]
related_skills: []
---

# 族实例 Host、HostFace、Subcomponent、Supercomponent 关系导航

## R — 原文 (Reading)

> 族实例对象有个 Host 属性，返回其主体图元。有些族实例对象没有主体图元，如桌子和其他家具，所以 Host 属性不返回任何内容。HostFace 属性获取对族实例的主体表面的参照。SuperComponent 属性返回族实例的父构件。GetSubComponentIds() 方法返回载入该族的族实例图元 ID。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.3 族实例（p162）

---

## I — 方法论骨架 (Interpretation)

族实例与外界有两类关系，各有专门的导航属性：

**宿主关系（垂直依附）：**
- `Host`：返回主体图元——门/窗 → 墙；基础 → 楼板/结构主体。**没有主体的实例（桌子、落地家具）返回 null**。
- `HostFace`：返回对主体**表面**的参照（Reference），用于基于面的定位。
- 注意语义边界：独立基础的 Host 恒为 null，取"自由宿主参数"要另走 `INSTANCE_FREE_HOST_PARAM`（楼板基础场景）。

**嵌套关系（构件包含）：**
- `GetSubComponentIds()`：返回嵌套在族里的子构件 ElementId 集合——餐桌族里的椅子。
- `SuperComponent`：返回父构件——椅子拿到餐桌。二者构成双向边。

**实战推论：**
1. 导航代码必须**容忍 null**——无主体、无父构件都是合法状态，不是 bug。
2. 嵌套族需要**递归展开**：从根实例反复 GetSubComponentIds → GetElement → 转 FamilyInstance，直到叶子。
3. 创建端复现宿主关系：创建门实例必须提供墙主体（NewFamilyInstance 的 host 参数），所以"找宿主"与"给宿主"是同一关系的两端。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建门必须给宿主
- **问题**: 创建门实例时报错或放置不正确。
- **方法论的使用**: 门是嵌入族，NewFamilyInstance 必须提供墙主体。
- **结论**: 宿主关系在创建端就确定，不是创建后可补的。
- **结果**: 提供墙主体后门正确嵌入，删除墙门也随之失效。

### 案例 2: 楼板基础的 Host 恒空
- **问题**: 独立基础的 Host 查询得到 null，以为是 bug。
- **方法论的使用**: 基础这类图元无主体概念，需走 INSTANCE_FREE_HOST_PARAM 等自由宿主参数。
- **结论**: 无主体是语义约定而非异常。
- **结果**: 正确分支处理，不再误判。

### 案例 3: 餐桌里的椅子
- **问题**: 想遍历一个"餐桌+椅子"的嵌套族里的每把椅子。
- **方法论的使用**: 对根实例 GetSubComponentIds() → GetElement → 判 FamilyInstance → 递归。
- **结论**: 嵌套族是递归结构，子构件本身可能再嵌子构件。
- **结果**: 成功枚举所有子构件，父方向用 SuperComponent 回溯。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 门窗要操作宿主墙（改墙厚、查墙参数），或反之从墙找嵌入的门窗。
2. 展开嵌套族（带椅餐桌、带设备的吊顶）统计子构件。
3. 写递归遍历族实例树的通用工具。
4. 判断实例是否为"自由放置"（无宿主）。

### 语言信号 (用户的话里出现这些就应激活)

- "门的宿主墙" / "get the host of this door"
- "嵌套族里的子构件" / "subcomponents of nested family"
- "GetSubComponentIds / SuperComponent"
- "HostFace / Host 返回 null"（"why is Host null"）
- "展开嵌套族" / "expand nested family"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定关系方向**
   - 宿主方向（上下依附）→ 步骤 2；嵌套方向（内部包含）→ 步骤 3。
   - 完成标准: 明确目标关系是 Host 系还是 Sub/Super 系。

2. **宿主导航**
   - 读 `Host`，先判 null；非 null 再按需读 `HostFace`。
   - 基础类自由放置构件 → 转走 INSTANCE_FREE_HOST_PARAM。
   - 完成标准: 每个 Host 读取都有 null 分支。

3. **嵌套导航（递归）**
   - 根实例 `GetSubComponentIds()` → 逐个 `GetElement` → 转 FamilyInstance 继续递归；用 `SuperComponent` 回溯。
   - 维护 visited 集合防循环引用。
   - 完成标准: 递归能到达全部叶子子构件，无死循环。
   - 判停条件: 若用户只是按类别过滤无宿主图元，无需本 skill，直接过滤即可。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只做类别/参数过滤，不需要实例间关系。
- 需要的是实例的类型/族归属（走三元导航），不是宿主。

### 作者在书中警告的失败模式

- Host 对无主体实例返回 null——不判空直接访问会抛异常或产生误导结果。
- 嵌套族子构件的 ID 集合只给 ID，必须先 GetElement 才能继续遍历。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：MEP 系统连接、更多宿主类型在后续版本扩展，但 Host/SubComponent 核心语义未变。

### 容易混淆的邻近方法论

- "HostFace" 是**表面参照**（Reference），用于基于面的创建/约束，与"主体图元"是两个粒度——别拿错。
- 嵌套族的子构件是"实例内实例"，与"族内的几何子元素"（Solid/Face）完全不同，后者属于几何模型范畴。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
