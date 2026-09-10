---
name: revit-worksharing-api-decision-framework
description: |
  写工作共享（worksharing）相关插件时调用：四步决策框架——检查 Document.IsWorkshared →
  EnableWorksharing 启用 → CheckoutElements/CheckoutWorksets 检出、RelinquishOwnership 放弃 →
  SetWorksetVisibility 可见性。性能原则：检索/检出/放弃都在一个大的调用中批量做（如 500 个
  ElementId 一次传入 CheckoutElements），不要循环逐个小调用。Trigger："检出"、"checkout"、
  "放弃所有权"、"工作集"、"relinquish"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.13.2–5.13.4（约p399-404）
tags: [worksharing, checkout, performance, revit-api]
related_skills:
  - slug: revit-enableworksharing-irreversible
    relation: depends-on
  - slug: revit-checkout-group-propagation
    relation: depends-on
---

# 工作共享下 API 操作决策框架

## R — 原文 (Reading)

> "WorksharingUtils 类可用于修改图元和工作集的所有权。CheckoutElements() 方法为当前用户获取指定图元的所有权，而 CheckoutWorksets() 方法获取尽可能多的指定工作的所有权。RelinquishOwnership() 方法让出当前用户所拥有的图元和工作集。"
>
> — 宦国胜，第5章 5.13.2–5.13.4（约p399–p404）

---

## I — 方法论骨架 (Interpretation)

工作共享插件的操作骨架是"所有权生命周期管理"，四步走：

1. **检查**：`Document.IsWorkshared` 判断文档是否已启用工作共享——决定后续路径是否存在。
2. **启用**（如需）：未启用时 `Document.EnableWorksharing(...)` 开启（注意其不可逆性，见专门 skill）。
3. **检出与放弃**：
   - `WorksharingUtils.CheckoutElements(doc, elementIds)` 批量获取图元所有权；
   - `CheckoutWorksets(doc, worksetIds)` 按工作集获取；
   - 操作完成后 `RelinquishOwnership(doc, options)` 一次性让出，供他人使用。
4. **可见性**：`SetWorksetVisibility` 控制工作集在视图中的显示。
- **性能原则（贯穿所有步骤）**："在一个大的调用中检索，而不是多个小调用"。原因：每次 API 调用都有内部开销（且检出涉及所有权协商），500 个图元循环单检出远慢于一次批量传入 ICollection。
- 心智模型：把"检出→操作→放弃"看成一个临界区，进出各一次批量调用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量检出 500 个图元

- **问题**: 工作共享工具要检出 500 个图元，逐个检出性能极差。
- **方法论的使用**: 收集全部 ElementId 到 ICollection，一次性传入 CheckoutElements；RelinquishOwnership 也在所有操作完成后一次调用。
- **结论**: "大调用优于小调用"是工作共享 API 的性能铁律。
- **结果**: 检出耗时从分钟级降到秒级，且所有权协商次数最小化。

### 案例 2: 编辑前确保所有权

- **问题**: 直接修改被他人检出的图元导致失败。
- **方法论的使用**: 修改前先 CheckoutElements（尽力获取"尽可能多"的指定图元），未获取成功的图元跳过并汇报。
- **结论**: 检出是写操作的前置条件，"尽力而为"语义要求代码处理部分失败。
- **结果**: 多用户环境下写操作不再随机失败，未检出图元被明确跳过。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写任何要修改工作共享文档图元的插件（批量编辑、标注、清理）。
2. 检出/放弃操作很慢，怀疑性能问题。
3. 多人协作时修改失败，要设计所有权流程。

### 语言信号 (用户的话里出现这些就应激活)

- "检出 / checkout / CheckoutElements"
- "放弃所有权 / relinquish ownership"
- "工作集 workset / IsWorkshared"

### 与相邻 skill 的区分

本 skill 与 `revit-checkout-group-propagation` 区分：那个讲检出会连带检出关联图元（如整组）的传递性行为，本 skill 讲检出的整体决策框架与批量性能原则。与 `revit-enableworksharing-irreversible` 区分：那个是启用操作的不可逆反例，本 skill 只把启用作为四步之一引用。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **检查工作共享状态**
   - `doc.IsWorkshared` 为 false 时决定：启用（走专门 skill 评估不可逆后果）或退出。
   - 完成标准: 状态检查在所有工作共享操作之前。
   - 判停条件: 文档未共享且用户不愿启用 → 不做检出逻辑，按单用户模式处理。

2. **批量检出**
   - 目标图元/工作集收集成集合，单次调用 CheckoutElements/CheckoutWorksets；处理"部分未获取"的返回语义。
   - 完成标准: 代码中无逐元素循环检出；未检出图元有明确处理分支。

3. **操作完成后一次性放弃**
   - 所有修改完成后单次 RelinquishOwnership（按 RelinquishOptions 配置范围）。
   - 完成标准: 放弃调用次数 = 1（每个操作临界区）；下一用户可立即获取所有权。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单机无共享文档——所有权概念不存在，直接操作即可（IsWorkshared=false 是退出信号）。
- 只读浏览——无需检出，读取不需要所有权（但注意打开配置，见打开工作共享文件 skill）。

### 作者在书中警告的失败模式

- 多次小调用检出/检索 → 性能塌方（书中明示）。
- 修改前不检出 → 他人持有时操作失败。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版 Revit Server/云协作（BIM 360/ACC）改变了中心文件形态，但 WorksharingUtils 所有权模型与批量原则仍适用。

### 容易混淆的邻近方法论

- QuickFilter vs SlowFilter 的批量检索性能分层（第1章）：同源的"大调用优于小调用"思想，作用在过滤器体系。

---

## 相关 skills

- revit-enableworksharing-irreversible（depends-on）：启用工作共享是不可逆操作，本 skill 四步中的启用步骤需先评估其后果。
- revit-checkout-group-propagation（depends-on）：检出会连带检出关联图元，本 skill 的检出决策依赖其传播语义。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
