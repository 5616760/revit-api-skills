---
name: revit-indexed-property-get-set
description: |
  当把书中的索引型属性（如 Element.Geometry）写成 C# 属性访问导致编译失败、或看到 get_/set_ 前缀方法名困惑时调用。规则：带参数的索引型'属性'必须写成 get_XXX(.
  ..)/set_XXX(...) 方法调用，Element.Geometry(options) 会编译报错。不适用：普通无参属性。trigger：get_Geometry、编译报错 CS1061、索引
  型属性、indexed property。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p020
tags: [syntax, csharp, indexed-property, compile-trap, revit-api]
related_skills: []
---

# 索引型 API 属性的 get_/set_ 前缀约定

## R — 原文 (Reading)

> 某些 Revit 平台 API 类属性是"索引型"的，在 API 帮助文件（RevitAPI.chm）中也称之为重载。例如，Element.Geometry 属性，本书中称之为属性，尽管在 C# 代码中可作为方法访问它们，但要在属性名称前加上前缀"get_"或"set_"。
>
> — 宦国胜, 第1章 1.1.7（约 p020）

---

## I — 方法论骨架 (Interpretation)

Revit API 有一部分"属性"其实带参数，是 COM/索引器遗留风格。它们在文档和书里被叫做 `Element.Geometry` 这样的属性名，但在 C# 里你不能按属性访问——必须把它们当作方法，并在名字前面加 `get_` 或 `set_` 前缀：

- 读 → `Element.get_Geometry(options)`，括号里传参数（如几何提取选项）。
- 写 → `Element.set_XXX(value)`（极少用，大多数索引型成员是只读的）。

两个最容易踩的坑：

1. 照书/照文档直接写 `Element.Geometry(options)` → 编译失败，因为 C# 里不存在这个带参数属性名。
2. 用 IDE 智能提示时看到 `get_Geometry` 觉得奇怪，以为写错了而删掉前缀 → 又编译失败。

识别方法：凡是在 RevitAPI.chm 里标注"Indexed"或"Overload"、且调用需要括号里传参的成员，基本都要走 `get_`/`set_` 前缀。几何提取 `Element.get_Geometry(Options)` 和参数检索 `Element.get_Parameter(BuiltInParameter)` 是全书出现率最高的两个例子。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 图元几何提取（第 1 章 1.5 / 第 3 章）
- **问题**: 要拿到墙的几何，书里说访问 `Element.Geometry` 属性。
- **方法论的使用**: 按约定改写为 `Element.get_Geometry(options)` 方法调用。
- **结论**: 带 Options 参数的成员必须走 get_ 前缀方法。
- **结果**: 代码在几何提取流程中正确编译并返回 GeometryElement。

### 案例 2: 参数检索（第 2 章）
- **问题**: 要按 BuiltInParameter 枚举取参数对象。
- **方法论的使用**: `element.get_Parameter(BuiltInParameter)` 而非属性访问。
- **结论**: 索引型成员在参数 API 中同样普遍。
- **结果**: 参数读写代码（s2 章节）编译通过。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 照着书或 RevitAPI.chm 抄代码，编译报 CS0117/CS1061，且报错位置有"属性名 + 括号"。
2. 看别人代码出现 `get_Geometry` 这类方法名，想确认是不是笔误。
3. 从 C#/Java 背景转来，习惯 `obj.Property(args)` 语法，需要转换思维。
4. 用 VB.NET 或 C++/CLI 时混淆了索引器与 get_/set_ 前缀的关系。

### 语言信号 (用户的话里出现这些就应激活)

- "Element.Geometry 编译报错 / 怎么写"
- "get_ 前缀是什么意思"
- "索引型属性 / 带参数的属性"
- "编译失败 CS1061 / CS0117"
- "indexed property / get_ prefix / how to access indexed API member"

### 与相邻 skill 的区分

- 与 `revit-internal-units-conversion` 的区别: 本 skill 是"语法形状"（怎么调用），units 是"数值语义"（取回的值是什么单位），两个都常出现在几何/参数代码里但问题不同。
- 与批次7参数专题 skill 的区别: 本 skill 讲 get_Parameter 这类成员的调用语法，参数专题讲参数类型与读写策略。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位报错或存疑的成员调用**
   - 完成标准: 确认该成员是带参数访问（括号里有参数），或文档标注为 Indexed/Overload。普通无参属性在此排除。

2. **改写为 get_/set_ 前缀方法**
   - 完成标准: 读操作改 `get_成员名(参数)`，写操作改 `set_成员名(参数)`；返回值类型与实参类型核对一致。若成员是只读（无 set_），向用户说明不能写。
   - 判停条件: 若改写后仍然编译失败，检查参数个数/类型（如 Options 是否 new 过），并核对是否查错了重载——跳到步骤 3。

3. **用 RevitAPI.chm 或 IDE 智能提示核对成员签名**
   - 完成标准: 在智能提示/文档中找到 `get_XXX` 条目并确认参数列表；编译通过。最后给用户一句规则总结：Revit 索引型"属性"一律按 `get_/set_` 方法调用。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 成员本身无参数（普通属性，如 `Element.Id`、`Element.Name`）——直接访问，不要加前缀。
- 泛型容器或 LINQ 场景（如 `ToElements()` 方法调用）——那是普通方法，不涉及前缀约定。

### 作者在书中警告的失败模式

- 直接照抄书里"属性名"而不加前缀 → 编译错误；此错误最常见于新手第一周。
- 把 `get_Geometry` 当成属性名继续加 `.` 访问子成员 → 类型不匹配。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0：较新版本已把大量索引型成员改为真正的 C# 索引器（如 `element[...]`），部分老代码在新版本会有不同写法；但 `get_Geometry`/`get_Parameter` 仍广泛存在。
- 书未系统列出"哪些是索引型"，需要靠 CHM 的 Indexed 标记识别，实践中靠报错反向定位。

### 容易混淆的邻近方法论

- `get_Parameter` 返回 `Parameter` 对象 ≠ `LookupParameter`（按名称查找）——前者是语法层，后者是语义层。
- 带前缀方法 ≠ 真正的 C# 索引器 `this[]`：Revit 索引型成员在托管包装里就是 `get_` 前缀的普通方法，别期待 `obj[...]` 下标语法可用。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
