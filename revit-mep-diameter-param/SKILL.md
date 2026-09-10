---
name: revit-mep-diameter-param
description: |
  原则 skill：Pipe.Diameter 等 MEP 几何属性只读，修改尺寸必须走内建参数（RBS_PIPE_DIAMETER_PARAM / RBS_DUCT_DIAMETER_PARAM / RBS_RECT_DUCT_WIDTH_PARAM / RBS_RECT_DUCT_HEIGHT_PARAM）。Trigger：改管径失败、Diameter 只读、批量调管径、resize duct/pipe。不适用于：创建管段、系统拓扑遍历。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.3.1.1 续（约p299）
tags: [revit-api, mep, built-in-parameter, diameter, principle]
related_skills:
  - slug: revit-builtin-parameter-semantics
    relation: depends-on
---

# Pipe.Diameter 属性只读，修改必须走 RBS_PIPE_DIAMETER_PARAM 内建参数

## R — 原文 (Reading)

> 在创建管道之后，如果希望更改管道直径，请获取 RBS_PIPE_DIAMETER_PARAM 内建参数。管道的 Diameter 属性是只读的，参见代码 4-27。
>
> — 宦国胜, 第4章 4.3.1.1 续（约p299）

---

## I — 方法论骨架 (Interpretation)

一条体现 Revit MEP 设计哲学的原则：**几何属性只读化，写操作收敛到内建参数**。

1. **现象**：`pipe.Diameter` 属性是只读的——编译期对它赋值就报错。风管尺寸同理。
2. **正确通道**：一切直径/宽高修改走内建参数：
   - 管道：`RBS_PIPE_DIAMETER_PARAM`
   - 风管（圆）：`RBS_DUCT_DIAMETER_PARAM`
   - 矩形风管：`RBS_RECT_DUCT_WIDTH_PARAM` / `RBS_RECT_DUCT_HEIGHT_PARAM`
   写法：`element.get_Parameter(BuiltInParameter.RBS_PIPE_DIAMETER_PARAM).Set(0.5)`。
3. **工程模式**：批量调径工具不要散落硬编码参数名——把四类尺寸参数封装成类型化枚举/辅助方法，按图元类型分派（管→直径、圆风管→直径、矩形风管→宽+高）。
4. **同构现象**：这不是孤例——RebarBarType/RebarHookType 等参数化数据、ThermalProperties 部分属性都是"只读属性 + 参数写入"的同一模式。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建后修改管道直径
- **问题**: 创建管道之后需要更改直径。
- **方法论的使用**: 按书中代码 4-27——获取 `RBS_PIPE_DIAMETER_PARAM` 内建参数并 Set 新值，而不是写 Diameter 属性（只读）。
- **结论**: 直径修改的唯一通道是参数 API。
- **结果**: 直径修改成功。

### 案例 2: "批量调整 MEP 管径"工具的防硬编码
- **问题**: 批量调径工具里避免散落的硬编码内建参数名。
- **方法论的使用**: 把四类尺寸参数（管径/风管直径/矩形宽/矩形高）封装成统一枚举，按图元类型分派——因为一切直径修改只能经参数 API，收敛封装无遗漏。
- **结论**: "只读属性 + 参数写入"的反差设计使封装成为唯一正解。
- **结果**: 工具对管、圆风管、矩形风管统一调径，无类型遗漏。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写批量调径/调风管尺寸工具（按流速/流量反算管径）。
2. 用户对 `pipe.Diameter` 赋值遇到编译错误，找正确写法。
3. 需要区分圆风管与矩形风管的参数组合（直径 vs 宽+高）。
4. 代码评审中检查尺寸修改是否走参数通道。

### 语言信号 (用户的话里出现这些就应激活)

- "Diameter 不能赋值 / 只读" / "Diameter is read-only / cannot set"
- "修改管径" / "change pipe diameter / resize duct"
- "RBS_PIPE_DIAMETER_PARAM" / "内建参数改尺寸"
- "矩形风管宽高" / "rectangular duct width height parameter"

### 与相邻 skill 的区分

- 与 `revit-builtin-parameter-semantics` 的关系：本 skill 聚焦 MEP 尺寸参数这组特定内建参数与“只读属性”反差，依赖该 skill 对内建参数语义的一般约束。
- 与 `revit-mep-curve-creation-entries` 的区别：该 skill 管“怎么建”；本 skill 专讲创建后改尺寸的参数通道。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认目标与参数名**
   - 按图元类型映射：管道→RBS_PIPE_DIAMETER_PARAM；圆风管→RBS_DUCT_DIAMETER_PARAM；矩形风管→RBS_RECT_DUCT_WIDTH_PARAM + RBS_RECT_DUCT_HEIGHT_PARAM。
   - 完成标准: 参数名已确定，不存在对 Diameter 属性赋值的代码。
2. **事务内写参数**
   - `element.get_Parameter(BuiltInParameter.XXX).Set(newValue)`（注意单位换算为英尺）。
   - 完成标准: Set 返回成功且读回值符合预期。
3. **批量封装（如需）**
   - 把参数选择封装为按类型分派的辅助方法，批量循环调用。
   - 完成标准: 批量工具对全部目标图元生效，无一因类型未覆盖而跳过/报错。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 创建管段本身——走创建入口 skill（创建时可带类型，不需事后改参数）。
- 修改系统类型/流体属性——不同的参数域。
- 假设一切 MEP 属性都只读——只有几何尺寸这类是"属性只读+参数可写"，读写前各自核对。

### 作者在书中警告的失败模式

- 直接给 `pipe.Diameter` 赋值——编译期即报错，不存在运行时侥幸。
- 矩形风管只改宽度漏改高度——尺寸参数成对出现。
- 硬编码参数字符串散落多处——维护时改名遗漏。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中部分属性开放了写通道或新增尺寸无关参数（如绝缘层），"只读属性+参数写入"总体格局延续但需逐属性核对。
- 单位换算（英尺 vs 毫米）书中提醒有限，迁移到公制项目时是常见坑。

### 容易混淆的邻近方法论

- 创建入口矩阵——创建时的类型/尺寸初始值与事后参数修改是两条路。
- 通用内建参数方法论——本原则是其在 MEP 尺寸域的实例。

---

## 相关 skills

- **revit-builtin-parameter-semantics**（内建参数（BuiltInParameter）的语义约束：只读属性走参数、跨语言稳定 · depends-on）— 直径写入走 RBS_PIPE_DIAMETER_PARAM 内建参数，依赖该 skill 的内建参数语义约束。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
