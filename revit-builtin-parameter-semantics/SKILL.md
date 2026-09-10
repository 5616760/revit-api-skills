---
name: revit-builtin-parameter-semantics
description: |
  当需要跨语言稳定地读写参数、判断图元属性，或遇到几何属性只读需走参数写入时调用。
  不适用于：共享参数检索（用 GUID）、族参数检索（无枚举）。
  关键 trigger 信号："BuiltInParameter 内建参数枚举"、"跨语言 multi-language"、"属性只读 read-only"、"改管道直径 change pipe diameter"、"Category.Name 判别失效"。
  核心认知：枚举句柄跨语言稳定，几何属性只读→写操作收敛到 BuiltInParameter；不能用 Category.Name 字符串做判别。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.3.1.1 续、4.2.1.5 等多处
tags: [builtin-parameter, enum-stability, cross-language, read-only, revit-api]
related_skills: []
---

# 内建参数（BuiltInParameter）的语义约束：只读属性走参数、跨语言稳定

## R — 原文 (Reading)

> 在创建管道之后，如果希望更改管道直径，请获取 RBS_PIPE_DIAMETER_PARAM 内建参数。管道的 Diameter 属性是只读的。
>
> — 宦国胜, 第4章 4.3.1.1 续、4.2.1.5 等多处

---

## I — 方法论骨架 (Interpretation)

BuiltInParameter 枚举有两个核心语义约束：

1. **跨语言稳定**：BuiltInParameter / BuiltInCategory 枚举的键（如 `RBS_PIPE_DIAMETER_PARAM`、`OST_PipeCurves`）在所有语言版本的 Revit 中恒定不变。而 `Category.Name` 在不同语言下返回本地化名称（如"Mechanical Equipment"在中文版是"机械设备"）。因此跨语言判别图元类别/读写参数时，必须用枚举而非 .Name 字符串。

2. **几何属性只读，写操作收敛到参数**：许多几何属性（如管道的 `Diameter`）是只读的，不能直接赋值。要修改这些属性，必须通过对应的 BuiltInParameter 参数来写。这是 Revit 将"几何表达"与"参数驱动"分离的设计——图元的几何状态由参数驱动，直接改属性会破坏数据一致性。

族参数没有 BuiltInParameter 枚举值，不能通过此路径检索。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 修改管道直径

- **问题**: 创建管道后需要更改直径，但 Diameter 属性只读
- **方法论的使用**: 识别"几何属性只读→写操作收敛到参数"——用 `element.get_Parameter(BuiltInParameter.RBS_PIPE_DIAMETER_PARAM)` 获取参数再赋值
- **结论**: 不能直接改 Diameter 属性，必须走 BuiltInParameter 参数写入
- **结果**: 管道直径成功修改

### 案例 2: 跨语言图元类别判别

- **问题**: 插件要同时跑在中文、英文、日文版 Revit 上，用 Category.Name 判别图元类别会静默失效
- **方法论的使用**: 用 BuiltInCategory 枚举（如 `BuiltInCategory.OST_PipeCurves`）而非 `Category.Name` 字符串——枚举键跨语言稳定
- **结论**: 字符串判别在多语言下失效，枚举句柄是唯一稳定路径
- **结果**: 插件在多语言 Revit 上正确判别图元类别

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开发跨语言 Revit 插件，需要稳定判别图元类别或读写参数
2. 尝试直接修改几何属性（如 Diameter）遇到只读错误
3. 用 Category.Name 字符串判别图元类型，在不同语言版本下失效
4. 需要知道某个几何属性的对应 BuiltInParameter 枚举值

### 语言信号 (用户的话里出现这些就应激活)

- "BuiltInParameter / 内建参数枚举"
- "跨语言 / multi-language / 中文英文日文 Revit"
- "属性只读 / read-only / 不能直接改"
- "改管道直径 / change pipe diameter"
- "Category.Name 判别失效 / category name localization"

### 与相邻 skill 的区分

- 与 `revit-parameter-index-lookup` 的区别：该 skill 关注四种检索入口的选择；本 skill 关注 BuiltInParameter 的跨语言稳定性与只读属性走参数的语义约束。
- 与 `revit-command-visibility-mode` 的区别：该 skill 关注命令可见性，与本 skill 的参数枚举语义无重叠。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定目标属性是只读几何属性还是可写参数**
   - 若是只读几何属性 → 查找对应的 BuiltInParameter 枚举
   - 若是参数 → 直接用枚举检索
   - 完成标准: 明确属性的只读/可写状态及对应枚举

2. **用 BuiltInParameter 枚举检索参数并写入**
   - `Parameter p = element.get_Parameter(BuiltInParameter.RBS_PIPE_DIAMETER_PARAM);`
   - `p.Set(newValue);`
   - 完成标准: 参数值成功修改
   - 判停条件: 若 get_Parameter 返回 null，检查枚举值是否正确、图元是否支持该参数

3. **跨语言判别用枚举而非字符串**
   - 用 `BuiltInCategory.OST_XXX` 而非 `Category.Name`
   - 完成标准: 判别逻辑在多语言下一致

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 共享参数——用 GUID 检索，BuiltInParameter 不覆盖共享参数
- 族参数——没有 BuiltInParameter 枚举值，只能按名或遍历检索

### 作者在书中警告的失败模式

- 用 Category.Name 字符串判别图元类别在多语言下静默失效
- 直接改只读几何属性会编译错误或运行时抛异常

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，BuiltInParameter 枚举覆盖范围可能在新版本中扩展
- 部分几何属性在后续版本中可能新增可写 API

### 容易混淆的邻近方法论

- BuiltInParameter（参数枚举，检索参数）vs BuiltInCategory（类别枚举，筛选图元）——前者改参数值，后者筛图元类别
- 几何属性只读（如 Diameter）vs 参数可写（如 RBS_PIPE_DIAMETER_PARAM）——同一概念的两个访问面

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
