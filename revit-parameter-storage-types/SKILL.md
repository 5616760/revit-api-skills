---
name: revit-parameter-storage-types
description: |
  当'参数读出来类型不对'、ElementId 参数为负数、或写参数报类型不匹配时调用。双分类：ParameterType（15 种语义类型）决定显示/单位/AsValueString；StorageT
  ype（5 种存储格式）决定 Get/Set 方法。陷阱：ElementId 负值（-1 自动检测等）非真实图元，解析前 if (id.Value >= 0)；写前检查 null 与 IsReadOn
  ly。不适用：单位换算。trigger：StorageType、ElementId 负数、parameter type mismatch、negative element id。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p081-083
tags: [parameters, storage-type, parameter-type, elementid, data-model]
related_skills:
  - slug: revit-command-result-undo
    relation: composes-with
---

# ParameterType 与 StorageType 双分类体系

## R — 原文 (Reading)

> ParameterType 枚举成员（表 2-5）：Length、Area、Volume、YesNo、Text、URL 等 15 种；StorageType 枚举（表 2-6）：String、ElementId、Double、Integer、None 5 种。
>
> — 宦国胜, 第2章 2.3.2 / 表 2-5 + 2.3.4 / 表 2-6（约 p081–p083）

---

## I — 方法论骨架 (Interpretation)

Revit 的参数有**两套并行的分类**，各管各的，别混：

1. **ParameterType（语义类型，15 种）**：描述"这个参数是什么意义"——Length、Area、Volume、YesNo、Text、URL、Material、Integer……它决定 UI 怎么显示、用什么单位、按什么分组，也是 `AsValueString`/`SetValueString` 做字符串格式化的依据。
2. **StorageType（存储格式，5 种）**：描述"底层怎么存"——String、ElementId、Double、Integer、None。它决定你调用哪对方法：`AsDouble/Set`、`AsString/Set`、`AsElementId/Set` 还是 `AsInteger/Set`。

读写参数的正确流程是**双层决策**：

- 先看语义（ParameterType）——判断这个参数意味着什么（如 YesNo 参数就是布尔语义）。
- 再看存储（StorageType）——选对应 Get/Set 方法（YesNo 通常按 Integer 存，0/1 表示）。

两个高频坑：

1. **写参数前置检查**：`parameter != null && !parameter.IsReadOnly` 再 `Set`，且值类型必须与 `StorageType` 匹配（给 Double 参数塞字符串直接抛异常）。
2. **ElementId 负值陷阱**：`StorageType.ElementId` 的负值（如 -1 自动检测、-2 梁中心、-3 梁顶等）表示**组合选项**，不是真实图元 ID。解析前必须 `if (id.Value >= 0)` 过滤，否则把 -2 当图元去查会得到 null。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 参数安全写入（代码 2-26）
- **问题**: 写参数前怎么避免运行时错误？
- **方法论的使用**: `if (parameter != null && !parameter.IsReadOnly)` 检查，再用 `StorageType` 比对值类型，不符则抛异常说明。
- **结论**: 写路径必须做类型与只读双重守卫。
- **结果**: 类型不匹配时抛清晰异常而非静默失败。

### 案例 2: AsValueString 依赖 ParameterType（2.3.5）
- **问题**: 同一个参数为什么能给出 "200"（字符串）和 -20（双精度）两种结果？
- **方法论的使用**: 按 ParameterType 用 AsValueString 走单位格式化；按 StorageType 用 AsDouble 拿裸值。
- **结论**: 两条取值路径由两套分类分别驱动。
- **结果**: 代码同时得到用户可读与程序可用两种形式。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 读参数报错"不能将字符串转换为双精度"之类——StorageType 没对上。
2. 看到 ElementId 参数值为 -2，以为是坏数据。
3. 写参数失败/无效，需要检查 IsReadOnly 和类型匹配。
4. 想知道 YesNo 参数到底怎么读写（按 Integer 0/1）。

### 语言信号 (用户的话里出现这些就应激活)

- "参数类型 / ParameterType / StorageType"
- "ElementId 是负数"
- "写参数报错 / 参数只读"
- "AsDouble 还是 AsString 还是 AsElementId"
- "parameter storage type / parameter type mismatch / negative element id / read-only parameter"

### 与相邻 skill 的区分

- 与 `revit-internal-units-conversion` 的区别: 本 skill 是"选对 Get/Set 方法与类型判断"，units 是"取到数值后的单位换算"——先类型后单位。
- 与批次7参数专题（共享参数/族参数）的区别: 本 skill 是参数读写的基础类型学，参数专题是参数的创建、分类与存储策略。
- 与 `revit-command-result-undo` 的区别: 写参数失败→返回 Failed 触发回滚（那是返回值语义），本 skill 负责让写入本身不失败。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定目标参数的双层类型**
   - 完成标准: 用 `parameter.Definition.ParameterType` 得到语义类型，`parameter.StorageType` 得到存储类型；把两者写出来并说明各自含义（如 "Length/Double：语义是长度、存储是双精度"）。

2. **选对 Get/Set 方法并加守卫**
   - 完成标准: 按 StorageType 选方法（Double→AsDouble/Set，String→AsString/Set，ElementId→AsElementId/Set，Integer→AsInteger/Set）；写路径加 `null && !IsReadOnly` 守卫且值类型与 StorageType 匹配。YesNo 参数按 Integer 0/1 处理。
   - 判停条件: 若是读 ElementId 参数，跳到步骤 3 处理负值。

3. **处理 ElementId 负值并验证**
   - 完成标准: `if (id.Value >= 0)` 才解析为真实图元；负值按"组合选项"处理（说明 -1 自动检测、-2 梁中心等语义）。运行验证读写正确。给用户一句速查：语义看 ParameterType、方法看 StorageType、负值先过滤。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单位换算（那是 units skill）——StorageType 定了但单位转换另算。
- 创建参数/共享参数/Extensible Storage（那是数据存储专题）。

### 作者在书中警告的失败模式

- 按 ParameterType 猜存储方法 → 类型不匹配抛异常（如把 Length 当 Integer 读）。
- 写前不检查 IsReadOnly → 写内置只读参数失败且不知为何。
- 把负 ElementId 当真实图元查询 → 返回 null 或查错对象。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：`ParameterType.Invalid` 与新版 `ForgeTypeId` 参数体系（2021+）差异大，旧双枚举仍是主流但新 API 用 `ForgeTypeId` 取代字符串类型。
- 书未列出全部负 ElementId 语义（如梁顶/梁底等结构选项），实操需结合 BuiltInParameter 文档核对。

### 容易混淆的邻近方法论

- `ParameterType`（语义/显示）vs `StorageType`（存储/方法）——最常见混淆点，记法："Type 管长什么样，Storage 管怎么取"。
- `ElementId` 参数（引用图元，负值=选项）vs `ElementId` 一般性引用（文档内图元 ID）——前者是参数值，后者是对象标识，别混。

---

## 相关 skills

- revit-command-result-undo：composes-with——写参数失败后按返回值语义返回 Failed 触发回滚，本 skill 负责让写入本身不失败，两者配套构成参数写入的完整防护。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
