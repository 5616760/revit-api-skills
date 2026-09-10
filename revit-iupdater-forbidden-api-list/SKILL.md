---
name: revit-iupdater-forbidden-api-list
description: |
  编写 IUpdater.Execute() 时防止调用禁入 API：ViewSheet.AddView、LoadFamily、
  NewAreaReinforcement/NewPathReinforcement、Save/SaveAs/Close、UpdaterRegistry 调用等一律禁止；
  即使不在清单内的方法，只要在图元间建立交叉引用也可能抛 ForbiddenForDynamicUpdateException。
  原则：修改优先于删除+重建。Trigger："更新器里调 LoadFamily/Save 报错"、
  "ForbiddenForDynamicUpdateException"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.6.2（约p359-360）
tags: [dmu, forbidden-api, counter-example, revit-api]
related_skills:
  - slug: revit-updater-registration-triggers
    relation: composes-with
---

# IUpdater Execute 中禁止调用的 API 清单

## R — 原文 (Reading)

> "当更新器执行时不可调用以下方法，因图元之间引入了交叉引用……ViewSheet.AddView()、Document.LoadFamily……在更新器 Execute()方法内部亦不允许这些调用。"
>
> — 宦国胜，第5章 5.6.2（约p359-360）

---

## I — 方法论骨架 (Interpretation)

更新器运行在事务收尾的敏感窗口，Revit 划出了两类禁入区。

- **显式禁入清单**（调用即异常）：
  - 交叉引用类：`ViewSheet.AddView()`、`Document.LoadFamily`、`NewAreaReinforcement()`、`NewPathReinforcement()` 等——它们会在图元间建立新引用，破坏事务收尾的一致性。
  - 文档生命周期类：`Save()`、`SaveAs()`、`Close()` 等——需要"事务-空闲"状态的方法在更新器内不可用。
  - 自指类：`UpdaterRegistry` 的调用（注册/注销更新器）在 Execute 内禁止。
- **隐式陷阱**：即使方法不在清单里，只要它在图元间建立交叉引用，也可能抛 `ForbiddenForDynamicUpdateException`——清单是下界不是上界。
- **工程原则**：修改既有图元优先于"删除旧图元+重建新图元"。删除+重建会破坏既有参照关系（尺寸标注、边界条件、连接），且重建动作更容易踩交叉引用禁区。
- 配套做法：把触发器范围配精确（见 `revit-updater-registration-triggers`），从源头减少 Execute 里的危险操作需求。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 更新器里加载族

- **问题**: 更新器逻辑需要用到新族，直觉调用 Document.LoadFamily。
- **方法论的使用**: LoadFamily 在显式禁入清单中（交叉引用类），调用必炸；应改为在 OnStartup 或外部命令中预加载，更新器只使用已加载的族。
- **结论**: 资源准备移出更新器，Execute 只做轻量修改。
- **结果**: 更新器不再抛异常，加载在安全上下文完成。

### 案例 2: "删除旧墙 + 重建一面墙"式更新

- **问题**: 更新器想通过删除+重建实现"换墙"。
- **方法论的使用**: 书中规则推导——重建会建立新交叉引用，即使不在清单内也可能抛 ForbiddenForDynamicUpdateException；且删除会破坏尺寸、连接等既有参照。
- **结论**: 修改优先于删除+重建。
- **结果**: 改为原地修改墙参数/类型后，参照关系保住、异常消失。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 编写更新器，设计 Execute 内要调用哪些 API。
2. 更新器抛 ForbiddenForDynamicUpdateException 或莫名异常，排查禁入调用。
3. 设计"重建式"更新逻辑（删除+新建）前的方案评审。

### 语言信号 (用户的话里出现这些就应激活)

- "更新器里能调 Save/LoadFamily 吗"（can I call Save/LoadFamily in updater）
- "ForbiddenForDynamicUpdateException"
- "更新器 删除重建"（updater delete and recreate）

### 与相邻 skill 的区分

- 与 `revit-updater-registration-triggers` 的关系：本 skill 讲 Execute 内禁入 API 清单；该 skill 讲注册与触发器配置，两者组合构成更新器安全使用闭环。
- 与 `revit-iupdater-execute-transaction-rules` 的区别：该 skill 讲事务规则；本 skill 讲具体 API 禁入清单，同是 Execute 安全边界但维度不同。
- 与 `revit-updater-failure-modes` 的区别：该 skill 讲运行期故障的 Revit 自动处置；本 skill 是编码期就该避开的静态禁区。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **对照禁入清单审计 Execute 代码**
   - 逐个检查：交叉引用类（AddView/LoadFamily/New*Reinforcement）、文档生命周期类（Save/SaveAs/Close）、UpdaterRegistry 调用。
   - 完成标准: 清单内 API 在 Execute 路径上零出现。
   - 判停条件: 发现禁入调用 → 进入第 2 步迁移，不尝试绕过。

2. **迁移禁入操作到合法上下文**
   - 资源加载→OnStartup/命令；文档保存关闭→Execute 之外由流程或事件驱动。
   - 完成标准: 每个被移除的调用都有新的执行位置说明。

3. **用"修改优先"原则复核重建式逻辑**
   - 完成标准: 存在"删除+重建"路径的，改为原地修改或论证为何必须重建（并接受参照丢失后果）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 外部命令/事件回调中的 LoadFamily、Save 等——禁入只针对更新器 Execute 上下文，别过度泛化。
- 纯读取场景——不建立交叉引用的读操作不受限。

### 作者在书中警告的失败模式

- 清单内 API 直接调用 → 异常。
- 清单外但建立交叉引用的调用 → 同样可能抛 ForbiddenForDynamicUpdateException（隐式陷阱）。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，清单为当时版本；新版禁入集合有变化，原则（交叉引用不可建）不变，具体方法以当前文档为准。

### 容易混淆的邻近方法论

- 事务三层级（Transaction/SubTransaction/TransactionGroup）：那是"怎么开事务"的规则；本 skill 是"哪些方法根本不能碰"的清单——先过清单再谈事务。

---

## 相关 skills

- **revit-updater-registration-triggers**（更新器注册与触发器配置 · composes-with）— 禁入 API 清单与注册/触发器配置同属更新器安全使用闭环。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
