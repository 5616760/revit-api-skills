---
name: revit-updater-registration-triggers
description: |
  让 IUpdater（DMU）生效时调用：UpdaterRegistry.RegisterUpdater() 注册 + AddTrigger() 配置
  触发器（元素过滤器 + GetChangeTypeXxx 变更类型双维度）。要点：OnStartup 注册/OnShutdown
  注销；GetChangeTypeAny() 只对"现有图元修改"触发，不含添加/删除；过滤器越精确越能避免无效
  执行与自触发。不适用于：事后通知（用 DocumentChanged）。Trigger："AddTrigger"、
  "ChangeTypeGeometry"、"更新器没触发"、"注册更新器"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.6.3（约p361-362）
tags: [dmu, updater-registry, triggers, revit-api]
related_skills:
  - slug: revit-iupdater-execute-transaction-rules
    relation: composes-with
  - slug: revit-filter-quick-slow-logical
    relation: composes-with
---

# 更新器注册与触发器配置

## R — 原文 (Reading)

> "更新器必须注册以便模型获得更改通知。应用程序级 UpdaterRegistry 类提供了注册/注销和操作更新器设置选项的能力……除调用 UpdaterRegistry.RegisterUpdater()方法以外，更新器还应通过 AddTrigger()方法添加一个或多个更新触发器。"
>
> — 宦国胜，第5章 5.6.3（约p361-362）

---

## I — 方法论骨架 (Interpretation)

更新器不是"实现接口就生效"，必须完成注册 + 触发器两件事，且触发器决定它何时被调用。

- 注册：`UpdaterRegistry.RegisterUpdater(updater, addInId)`（或带文档参数的重载）；注销在 `OnShutdown` 中调用 `UnregisterUpdater(updaterId)`——与事件注册同样的生命周期配对。
- 触发器双维度：
  1. **范围**（哪些图元）：ElementClassFilter / 类过滤器限定类别（如 `typeof(Wall)`）。
  2. **变更类型**（什么变化）：`Element.GetChangeTypeAny/Geometry/Parameter(...)` 系列静态方法。
- 关键语义陷阱：`GetChangeTypeAny()` 只对**现有图元的修改**触发，不覆盖添加/删除——要监听增删必须分别添加触发器。
- 精确性原则：过滤越精确 → 无效执行越少 → 自触发风险越低 → 性能越好。
- 典型配置形态：`AddTrigger(updaterId, wallFilter, Element.GetChangeTypeGeometry())` = "墙的几何变化时触发"。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 只在"墙的几何变化"时触发

- **问题**: 更新器不想被任何参数变化打扰，只关心墙几何改变。
- **方法论的使用**: 双维度配置——`ElementClassFilter(typeof(Wall))` 限定范围 + `Element.GetChangeTypeGeometry()` 限定变更类型。
- **结论**: 精确触发器把无关变更全部挡在门外。
- **结果**: 更新器只在墙几何真正变化时执行，避免自触发与性能浪费。

### 案例 2: 需要覆盖添加/删除/修改三种事件

- **问题**: 用一个 GetChangeTypeAny() 想覆盖所有变更，结果新增/删除墙时不触发。
- **方法论的使用**: 按书中语义——Any 不含增删——为三类变更分别 AddTrigger（或分别用对应 ChangeType）。
- **结论**: "Any"不是"全部"，是"任何现有图元的修改"。
- **结果**: 三触发器配置后，增、删、改全部被正确捕获。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写好 IUpdater 但从不被触发，排查注册/触发器配置。
2. 更新器触发太频繁或自己触发自己（A 改墙又触发 A）。
3. 需要监听"图元被添加/删除"的场景发现 Any 失效。

### 语言信号 (用户的话里出现这些就应激活)

- "RegisterUpdater / AddTrigger"
- "GetChangeTypeGeometry / GetChangeTypeAny / ChangeTypeParameter"
- "更新器不触发 / 触发太频繁"（updater not firing / firing too often）

### 与相邻 skill 的区分

- 与 `revit-iupdater-execute-transaction-rules` 的关系：本 skill 管“怎么注册、监听什么”；该 skill 管“Execute 内部怎么写、事务规则是什么”，入口与身体互补。
- 与 `revit-filter-quick-slow-logical` 的关系：更新器触发器（Trigger）依赖 ElementFilter，复用该 skill 的过滤器选型。
- 与 `revit-event-registration-two-steps` 的区别：事件注册监听 Revit 通知；更新器注册是“声明介入修改”，带过滤器/变更类型两个配置维度。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **注册更新器（OnStartup/OnShutdown 配对）**
   - `UpdaterRegistry.RegisterUpdater(updater, addInId)` + 记录返回的 UpdaterId；注销写在 OnShutdown。
   - 完成标准: 注册/注销成对，UpdaterId 被保存供 AddTrigger 使用。

2. **配置触发器（范围 × 变更类型）**
   - 明确列出：监听哪些类别的图元、哪种变更；增/删/改分别需要时逐个 AddTrigger。
   - 完成标准: 每个触发器都是"过滤器 + ChangeType"的显式组合；没有用 Any 偷懒覆盖增删。
   - 判停条件: 若需求是"任何文档变化都要知道"（审计类），改用 DocumentChanged 事件，不需要更新器。

3. **验证触发行为**
   - 完成标准: 手工测试增/删/改/无关参数修改四类操作，只在预期场景触发。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 纯通知/日志需求 → DocumentChanged 事件足够，更新器是为"修改"而生的。
- 跨文档监听 → 更新器作用域是文档内模型变更。

### 作者在书中警告的失败模式

- 以为 GetChangeTypeAny 覆盖添加/删除 → 增删时更新器沉默。
- 触发器过宽 → 频繁无效执行、自触发循环（见 `revit-updater-failure-modes`）。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版触发器 API 有扩展（如 GetChangeTypeElementDeletion 细化、ChangePriority），配置思路不变。

### 容易混淆的邻近方法论

- QuickFilter/SlowFilter 性能分层（第1章）：过滤器语义同源——本 skill 的范围维度就是复用元素过滤器体系。

---

## 相关 skills

- **revit-iupdater-execute-transaction-rules**（IUpdater Execute 方法约束与事务规则 · composes-with）— 注册与触发器是入口，Execute 事务规则是身体，二者组合构成更新器完整用法。
- **revit-filter-quick-slow-logical**（ElementFilter 三类策略选择：QuickFilter / SlowFilter / LogicalFilter · composes-with）— 触发器配置使用 ElementFilter，复用该 skill 的 QuickFilter/SlowFilter/LogicalFilter 选型。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
