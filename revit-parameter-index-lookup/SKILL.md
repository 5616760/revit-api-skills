---
name: revit-parameter-index-lookup
description: |
  当需要在 Revit 图元上检索参数且面临多语言环境、共享参数/族参数/内置参数混用时调用。
  不适用于：已知参数名且单语言项目、需要遍历全部参数做批量操作。
  关键 trigger 信号："参数查不到 parameter not found"、"多语言 multi-language"、"共享参数 GUID shared parameter GUID"、"BuiltInParameter 内建参数枚举"。
  核心决策：按参数来源选检索入口——共享参数用 GUID、内置参数用 BuiltInParameter、族参数只能按名遍历。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第2章 2.3.2 (约p081-082)
tags: [parameter-retrieval, guid, builtin-parameter, multi-language, revit-api]
related_skills:
  - slug: revit-builtin-parameter-semantics
    relation: composes-with
---

# 参数定义与检索决策树：Definition / Parameter[String]/GUID 三种索引

## R — 原文 (Reading)

> Definition 对象描述了数据类型、名称和其他参数细节。InternalDefinition 表示全部存在 Revit 数据库中的所有种类的定义。ExternalDefinition 表示外部存储在硬盘共享参数文件中的定义。
>
> — 宦国胜, 第2章 2.3.2 (约p081-082)

---

## I — 方法论骨架 (Interpretation)

Revit 提供四条参数检索路径，稳定性递增：

1. **名称检索**：`element.LookupParameter("参数名")` 按字符串匹配，最直观但参数名随 Revit 语言版本本地化，中文版与英文版名称不同，多语言项目会静默失效。
2. **BuiltInParameter 枚举**：`element.get_Parameter(BuiltInParameter.WALL_TOP_OFFSET)` 用枚举句柄检索，跨语言稳定，但只覆盖内置参数，族参数没有对应的枚举值。
3. **GUID 检索**：`element.get_Parameter(new Guid("..."))` 用共享参数的全局唯一标识检索，跨语言、跨会话、跨模型稳定，是共享参数的最佳检索方式。
4. **Definition 对象**：区分 InternalDefinition（数据库内定义）与 ExternalDefinition（硬盘共享参数文件中的定义），用于定义级别的元数据访问。

选哪条路径取决于参数来源：共享参数首选 GUID，内置参数用 BuiltInParameter，族参数只能按名或遍历。写参数前须检查 StorageType，单位安全用 AsValueString/SetValueString。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 多语言项目共享参数检索

- **问题**: 中文/英文 Revit 混用环境下，按名称检索共享参数在不同语言版本下返回 null
- **方法论的使用**: 识别参数来源为共享参数，选择 GUID 路径而非名称路径——共享参数有唯一 GUID，跨语言稳定
- **结论**: `element.get_Parameter(new Guid("..."))` 是唯一可靠路径，不能用 `LookupParameter("本地化名称")`
- **结果**: 插件在中/英/日文 Revit 上均能正确检索共享参数

### 案例 2: 族参数检索的盲区

- **问题**: 尝试用 BuiltInParameter 枚举检索族参数，始终返回 null
- **方法论的使用**: 识别参数来源为族参数——BuiltInParameter 只覆盖内置参数，族参数未公开枚举值
- **结论**: 族参数只能走 LookupParameter（按名）或遍历 Parameters 集合
- **结果**: 避免了对族参数误用 BuiltInParameter 导致的 null 陷阱

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开发跨语言 Revit 插件，需要稳定检索共享参数而不受语言版本影响
2. 参数检索返回 null 或错误值，需要诊断检索路径是否选错
3. 需要区分内置参数、共享参数、族参数并选择正确的检索入口
4. 使用 ElementParameterFilter 构造参数过滤器，需要参数 ID

### 语言信号 (用户的话里出现这些就应激活)

- "参数查不到 / parameter not found / LookupParameter returns null"
- "多语言 / multi-language / 中文英文 Revit"
- "共享参数 GUID / shared parameter GUID"
- "BuiltInParameter / 内建参数枚举"

### 与相邻 skill 的区分

- 与 `revit-builtin-parameter-semantics` 的关系：本 skill 是四种检索入口的选择决策树；该 skill 补充枚举的只读属性约束与跨语言稳定性语义，组合覆盖参数访问。
- 与 `revit-data-storage-paths` 的区别：该 skill 关注数据存储选型（共享参数 vs 可扩展存储）；本 skill 关注已有参数的检索路径。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断参数来源，选择检索入口**
   - 共享参数 → GUID 路径
   - 内置参数 → BuiltInParameter 枚举
   - 族参数 → LookupParameter 按名或遍历 Parameters
   - 完成标准: 明确参数属于哪一类来源

2. **按选定入口检索参数对象**
   - GUID: `element.get_Parameter(new Guid("..."))`
   - BuiltInParameter: `element.get_Parameter(BuiltInParameter.XXX)`
   - 名称: `element.LookupParameter("参数名")`
   - 完成标准: 返回非 null 的 Parameter 对象
   - 判停条件: 若族参数且 LookupParameter 返回 null，检查名称拼写与语言版本后停止

3. **读写前做类型与单位安全检查**
   - 读前检查 `parameter.StorageType`；用 `AsValueString/SetValueString` 做单位安全转换
   - 完成标准: 参数值正确读写且单位匹配

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 纯内置参数且无需跨语言——直接用 BuiltInParameter 即可，无需决策树
- 需要遍历所有参数做批量操作——直接遍历 Parameters 集合，无需逐个检索

### 作者在书中警告的失败模式

- LookupParameter 按名称在多语言下静默返回 null（不报错，极易误判为参数不存在）
- 对族参数误用 BuiltInParameter 返回 null

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增参数检索 API（如 GetOrderedParameters），枚举覆盖范围也可能扩展

### 容易混淆的邻近方法论

- BuiltInParameter（参数枚举）vs BuiltInCategory（类别枚举）——前者检索参数，后者筛选图元
- Definition（定义对象）vs Parameter（参数实例）——前者描述元数据，后者持有实际值

---

## 相关 skills

- **revit-builtin-parameter-semantics**（内建参数（BuiltInParameter）的语义约束：只读属性走参数、跨语言稳定 · composes-with）— 检索到 BuiltInParameter 后如何读写语义由该 skill 提供，二者组合覆盖参数访问全链路。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
