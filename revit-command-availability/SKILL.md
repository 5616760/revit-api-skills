---
name: revit-command-availability
description: |
  当要让按钮'选中墙才可点、否则灰显'、或写 IExternalCommandAvailability/IsCommandAvailable、或在 .addin 配 AvailabilityClassN
  ame 时调用。实现 IsCommandAvailable 用 CategorySet 判断选集，返回 true 可用/false 灰显；AvailabilityClassName 接线。动态机制，区
  别于 VisibilityMode 静态显隐。不适用：按文档类型显隐。trigger：按钮灰显、选中墙才可用、command availability、gray out button。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p035-036
tags: [ribbon-button, availability, category-filter, ui-control]
related_skills:
  - slug: revit-plugin-entry-types
    relation: depends-on
---

# 命令可用性控制：IExternalCommandAvailability

## R — 原文 (Reading)

> IExternalCommandAvailability 接口可控制一个外部命令按钮是否可用。IsCommandAvailable 接口方法传递应用以及一组与 Revit 所选项类别相匹配的类别到应用。典型用途是检查所选的类别，看它们是否满足命令运行条件。
>
> — 宦国胜, 第1章 1.3.2（约 p035–p036）

---

## I — 方法论骨架 (Interpretation)

想让按钮"懂眼色"——只在满足条件时能点——Revit 提供两个机制，本 skill 讲**动态**的那一个：

- 类实现 `IExternalCommandAvailability`，实现 `IsCommandAvailable(UIApplication application, CategorySet selectedCategories)`。
- 每当选择集变化，Revit 调用该方法：返回 `true` → 按钮可用；`false` → 灰显。
- `CategorySet` 就是当前选中图元所属类别的集合，典型逻辑是"检查类别是否包含墙面"之类。

注册方式：这个可用性类的全名（命名空间.类名）填进 `.addin` 清单对应命令条目的 `AvailabilityClassName` 字段——按钮和可用性判断器由此绑定。

关键区分：**动态 vs 静态**。`IsCommandAvailable` 基于实时选集动态判断（选择变了按钮状态跟着变）；而 `VisibilityMode`/`Discipline` 是静态声明（按文档类型决定整个按钮的显隐）。选墙按钮用前者，建模/分析类按钮的文档归属用后者。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 代码 1-13 墙分析按钮
- **问题**: 一个"墙分析"命令，希望没选中墙时按钮灰掉。
- **方法论的使用**: 实现 `IsCommandAvailable`，检查 `CategorySet` 是否含 `OST_Walls`；空选集或至少选中一面墙时返回 true。
- **结论**: 用类别集合判断，条件灵活且实时。
- **结果**: 按钮随选择集变化自动可用/灰显。

### 案例 2: .addin 的 AvailabilityClassName 注册（1.3.4）
- **问题**: 可用性判断类怎么和命令绑定？
- **方法论的使用**: 在 .addin 的 `Command` 条目中填写 `AvailabilityClassName`。
- **结论**: 清单文件负责接线，无需代码注册。
- **结果**: 命令与可用性控制按清单绑定生效。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 按钮想让"选中墙/选中管道/选中族实例"才可点，否则灰显。
2. 实现 `IExternalCommandAvailability` 时不知道方法签名或返回值逻辑。
3. .addin 里配了按钮但按钮永远可点，怀疑 `AvailabilityClassName` 没生效。
4. 分不清"按钮灰显"应该用 Availability 还是 VisibilityMode。

### 语言信号 (用户的话里出现这些就应激活)

- "按钮没选中东西就不让点 / 灰显"
- "选中墙才能用这个命令"
- "IsCommandAvailable / AvailabilityClassName"
- "命令可用性控制"
- "enable button only when wall selected / command availability / gray out button"

### 与相邻 skill 的区分

- 与 `revit-ribbon-control-types` 的区别: 本 skill 控制按钮的可用状态，ribbon-control-types 决定用什么控件类型（按钮/下拉/单选组）。
- 与 `revit-plugin-entry-types` 的区别: 本 skill 是命令增强能力（可用性），entry-types 是入口选型；AvailabilityClassName 是清单字段的扩展。
- 与批次3过滤专题的区别: 本 skill 用 CategorySet 做 UI 状态判断，过滤专题用 FilteredElementCollector 在代码里检索图元。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **写可用性类**
   - 完成标准: 新建类实现 `IExternalCommandAvailability`，`IsCommandAvailable(UIApplication, CategorySet)` 返回 bool，条件逻辑明确（如 `selectedCategories.Contains(BuiltInCategory.OST_Walls)` 或空选集放行）。编译通过。

2. **在 .addin 中接线**
   - 完成标准: 对应命令条目的 `AvailabilityClassName` 填为该类全名；核对拼写与命名空间。
   - 判停条件: 若按钮是通过代码创建（PushButtonData 无 Availability 字段）、或需求是按文档类型静态显隐，停——分别指路 .addin 注册方式与 VisibilityMode。

3. **验证状态联动**
   - 完成标准: 重启 Revit，测试选集变化时按钮可用/灰显正确切换。给用户区分提示：这是动态控制；若需求是"按文档类型显隐"（静态），指路 VisibilityMode 而非本 skill。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 按钮按"当前文档是项目还是族"这类静态条件显隐 → 用 `VisibilityMode`（VisibilityMode/Discipline）。
- 想控制按钮以外的控件（如整个面板）显隐 → 没有对等 API，需换设计思路。

### 作者在书中警告的失败模式

- `AvailabilityClassName` 拼错/未注册 → 按钮永不灰显（默认可点），逻辑形同虚设。
- `IsCommandAvailable` 里做重活（如遍历文档）→ 每次选择集变化都执行，拖慢 UI。
- 返回逻辑与命令实际前提不符 → 按钮可点但执行即报错，可用性判断应前置校验。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版 Ribbon 状态控制更丰富（按钮文本/提示的动态更新），`Availability` 机制本身稳定。
- 书未讨论多命令共享同一可用性类的场景，以及 `CategorySet` 在未选择任何图元时的空集行为。

### 容易混淆的邻近方法论

- `IsCommandAvailable`（动态、每选一次调一次）vs `VisibilityMode`（静态、声明式配置）——目标是同一个"按钮可点性"，机制完全不同。
- `CategorySet`（选中项的类别）vs `ElementCategoryFilter`（检索过滤器类别）——一个是 UI 状态输入，一个是查询条件。

---

## 相关 skills

- revit-plugin-entry-types：depends-on——本 skill 讲命令可用性增强，前提是先理解插件入口类型；AvailabilityClassName 是清单里 Command 条目的扩展字段。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
