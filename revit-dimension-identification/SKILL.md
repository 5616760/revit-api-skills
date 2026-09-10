---
name: revit-dimension-identification
description: |
  识别/遍历项目尺寸（Dimension）时。API 无直接子类型判别，须组合推断：OST_Dimensions 过滤 → Dimension.Curve 形状（Line=线性、Arc=弧长/角度、null=径向）× References 数量；径向 Curve 为 null，遍历必须先判空。SpotDimension 单独处理。创建限制：NewDimension() 仅可创建线性尺寸。不适用于：注释图元其他类型。
  Trigger："尺寸按类型分类"、"径向尺寸 Curve 为 null"、"怎么辨别五种尺寸"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.6.1（约 p202–p205）
tags: [dimension, dimension-type-inference, ost-dimensions, curve-shape, linear-only]
related_skills:
  - slug: revit-reference-stable-handle
    relation: composes-with
---

# 尺寸识别与遍历框架（含 NewDimension 只能创建线性尺寸的限制）

## R — 原文 (Reading)

> 所有永久性尺寸的内建类别都是 OST_Dimensions。API 没有简单的方法来辨别以上五种尺寸。除半径和直径尺寸外，每个尺寸都有个尺寸线。尺寸线可从 Dimension.Curve 属性获取，该属性始终是未绑定的。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.6.1 节（约 p202–p204）

---

## I — 方法论骨架 (Interpretation)

Revit 有五种永久尺寸（线性、弧长、角度、半径、直径），但 API 没有提供"直接告诉我它是哪种"的便捷方法——你必须自己组合证据推断。

推断框架由两条线索交叉构成。第一条是尺寸线（Dimension.Curve）的形状：线性尺寸的曲线是 Line，弧长/角度尺寸是 Arc，而半径/直径尺寸没有尺寸线——Curve 是 null。第二条是 References：被标注对象的引用集合。把"Curve 形状（Line/Arc/null 三分支）× References 数量"组合起来，就能把五种类型一一对上。由此还得到一个硬规则：径向尺寸 Curve 为 null，任何依赖尺寸线的遍历代码必须先判空，否则直接空引用。SpotDimension（高程点标注）是另一类，按高程点单独处理。

创建侧的限制同样硬：NewDimension() 只能创建线性尺寸，带 DimensionType 参数的重载很少使用——别指望 API 帮你造出径向/角度尺寸。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 五种尺寸的分类推断（3.6.1）
- **问题**: 把项目所有尺寸按子类型分类统计。
- **方法论的使用**: OST_Dimensions 过滤 → Curve 形状 × References 数量组合推断。
- **结论**: 组合推断是唯一可行的判别路径。
- **结果**: 反向推知径向尺寸 Curve 为 null，遍历需先判空。

### 案例 2: 族参数绑定线性尺寸（3.4，约 p178）
- **问题**: 族文件中给模型线加尺寸并绑定"宽度"族参数。
- **方法论的使用**: NewDimension + Reference 创建线性尺寸。
- **结论**: 线性尺寸是创建 API 支持的全部。
- **结果**: 参数化族语境复用同一 API（见 s2b-c08）。

### 案例 3: 创建限制（3.6.1）
- **问题**: 需要创建非线性的尺寸。
- **方法论的使用**: 确认 NewDimension 只支持线性，带 DimensionType 的重载很少使用。
- **结论**: 创建能力有限，识别与创建互补。
- **结果**: 识别是主要程序化场景。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 统计/分类项目中的所有尺寸标注（出图检查、规范校验）。
2. 遍历尺寸时对径向尺寸 Curve 为 null 的判空处理。
3. 识别 SpotDimension 与普通尺寸。
4. 尝试用 NewDimension 创建尺寸时了解能力边界。

### 语言信号 (用户的话里出现这些就应激活)

- "把所有尺寸按类型分类" / "classify all dimensions by type"
- "径向尺寸的 Curve 是 null" / "radial dimension curve is null"
- "API 怎么辨别五种尺寸" / "how to tell dimension subtypes apart"
- "NewDimension 能创建什么尺寸" / "what can NewDimension create"

### 与相邻 skill 的区分

- 与 `revit-reference-stable-handle`：本 skill 用 References 数量做尺寸分类推断，其 References 语义依赖后者的稳定句柄框架，组合使用。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **过滤尺寸图元**
   - 完成标准: 用 BuiltInCategory.OST_Dimensions 过滤拿到尺寸集合；识别出 SpotDimension 单独处理。

2. **读取并判空**
   - 完成标准: 逐个读 Dimension.Curve；先判 null（径向），再分 Line/Arc 分支；读 References 数量与内容。

3. **组合推断子类型**
   - 完成标准: 用"Curve 形状 × References 数量"对照五种子类型给出分类；需要创建时确认只做线性（NewDimension 限制）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 创建复杂尺寸类型（角度/径向）：API 不支持，只能线性。
- 处理详图线、文字等注释图元：不是 Dimension。

### 作者在书中警告的失败模式

- 依赖尺寸线遍历却遇到径向尺寸：Curve 为 null，不判空必崩。
- 找"简单判别方法"却不存在：唯一路径是组合推断，别浪费时间找捷径。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：OST_Dimensions、Curve 语义在新版本保持；新版本 Dimension 类型有所扩展（如多段尺寸、等距约束），但五种子类型的推断框架仍适用；Curve 未绑定的契约不变。

### 容易混淆的邻近方法论

- "无尺寸线"（径向）与"Curve 异常"：径向尺寸 Curve 为 null 是正常契约，不是数据损坏。
- SpotDimension 与普通 Dimension：前者是独立类别，按高程点处理，别混进五种子类型推断。

---

## 相关 skills

- **revit-reference-stable-handle**（composes-with）：本 skill 用 References 数量做尺寸分类推断，其 References 语义依赖后者的稳定句柄框架，组合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
