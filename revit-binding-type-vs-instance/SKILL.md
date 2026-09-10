---
name: revit-binding-type-vs-instance
description: |
  当需要将共享参数绑定到图元类别且面临类型绑定 vs 实例绑定时调用。
  不适用于：可扩展存储、族参数（走 FamilyManager）。
  关键 trigger 信号："绑定共享参数 bind shared parameter"、"类型参数 type parameter"、"实例参数 instance parameter"、"所有实例共享 same value all instances"、"BindingMap Insert"。
  核心决策：值需所有同类型实例共享 → TypeBinding；值需每实例独立 → InstanceBinding。同一 Definition 不能同时绑两种。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1.3 (约p319-322)
tags: [binding, type-binding, instance-binding, shared-parameter, revit-api]
related_skills:
  - slug: revit-binding-insert-silent-fail
    relation: composes-with
---

# 共享参数绑定（Binding）的两种模式：TypeBinding vs InstanceBinding

## R — 原文 (Reading)

> TypeBinding 对象用于将属性绑定于 Revit 类型……属性是由类型绑定所识别出的全部实例所共享。更改某个类型的参数会影响到所有相同类型的实例。InstanceBinding 对象表示在某个参数定义与某类别的实例参数之间的绑定……更改任意某个实例的参数，不会导致任何其他实例中参数值的变化。
>
> — 宦国胜, 第5章 5.1.3 (约p319-322)

---

## I — 方法论骨架 (Interpretation)

共享参数绑定到图元类别时有两种模式，决定参数值是存在类型上还是实例上：

**TypeBinding（类型绑定）**：参数值绑定到图元类型（如墙类型 W1），该类型的所有实例共享同一个值。改任何一面墙的该参数 = 改类型参数，全部实例同步变化。适合"成本""防火等级"等同类实例共享的属性。

**InstanceBinding（实例绑定）**：参数值绑定到图元实例，每个实例有独立的值。改某个实例不影响其他实例。适合"出厂日期""安装位置"等每实例独立的属性。

关键约束：同一 Definition 不能既绑 Instance 又绑 Type。第二次 Insert 不会抛异常而是静默返回 false，开发者必须检查返回值判断是否绑定成功。这与族编辑器中类型参数/实例参数的二分呼应——族参数在 FamilyManager 中也做 type/instance 严格区分。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 墙类型成本参数绑定

- **问题**: 墙类型 W1 有 50 面实例，要加"成本"参数，所有 W1 墙成本相同，改一次全改
- **方法论的使用**: 值需所有同类型实例共享 → TypeBinding。改任何一面墙的成本 = 改该类型参数值，全部 W1 实例同步变化
- **结论**: 选 TypeBinding，维护成本低
- **结果**: 改一处即全改，批量维护效率高

### 案例 2: 每实例独立参数的误绑

- **问题**: "出厂日期"参数误用 TypeBinding，导致所有实例出厂日期相同
- **方法论的使用**: 值需每实例独立 → InstanceBinding。出厂日期因实例而异，必须实例绑定
- **结论**: 选 InstanceBinding
- **结果**: 每面墙有独立出厂日期值

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 将共享参数绑定到图元类别，需要决定用类型绑定还是实例绑定
2. 参数值在所有同类型实例间共享 vs 每实例独立值
3. BindingMap.Insert 返回 false，需要诊断绑定失败原因
4. 族参数管理中区分类型参数与实例参数

### 语言信号 (用户的话里出现这些就应激活)

- "绑定共享参数 / bind shared parameter / BindingMap"
- "类型参数 / type parameter / TypeBinding"
- "实例参数 / instance parameter / InstanceBinding"
- "所有实例共享 / same value all instances"

### 与相邻 skill 的区分

- 与 `revit-binding-insert-silent-fail` 的关系：本 skill 是类型/实例绑定的选型决策；该 skill 讲共享参数不能同时绑 Instance 和 Type 的静默失败与防御，二者组合使用。
- 与 `revit-data-storage-paths` 的区别：该 skill 是共享参数 vs 可扩展存储选型；本 skill 是共享参数内部的绑定模式选型。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断参数值的共享需求**
   - 所有同类型实例共享同一值 → TypeBinding
   - 每实例独立值 → InstanceBinding
   - 完成标准: 明确值的共享粒度

2. **构造 Binding 对象并 Insert**
   - `TypeBinding binding = new TypeBinding(category);` 或 `InstanceBinding binding = new InstanceBinding(category);`
   - `bool success = document.ParameterBindings.Insert(definition, binding);`
   - 完成标准: Insert 返回 true
   - 判停条件: 若返回 false，说明该 Definition 已有绑定（不能同时绑两种），检查 `bindingMap.Contains(definition)` 后停止

3. **验证绑定生效**
   - 取该类别的实例/类型，确认参数出现在属性对话框中
   - 完成标准: 参数可见且值正确

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 可扩展存储——不走 BindingMap，用 Schema/Entity 机制
- 族参数——走 FamilyManager.AddParameter，不在项目文档的 BindingMap 中绑定

### 作者在书中警告的失败模式

- 同一 Definition 不能同时绑 Instance 和 Type——第二次 Insert 静默返回 false（不抛异常）
- 必须检查 Insert 的 bool 返回值，否则会误以为绑定成功

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，BindingMap 的 API 签名可能在新版本中变化

### 容易混淆的邻近方法论

- 项目文档的 BindingMap（类别绑定）vs 族文档的 FamilyManager（族参数绑定）——前者绑到类别，后者绑到族定义
- TypeBinding（类型绑定）vs 类型参数（FamilyManager 中的 type parameter）——概念相似但 API 入口不同

---

## 相关 skills

- **revit-binding-insert-silent-fail**（共享参数不能同时绑 Instance 和 Type（Insert 静默返回 false） · composes-with）— 绑定模式选型后可能踩 Insert 静默失败陷阱，该 skill 提供防御。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
