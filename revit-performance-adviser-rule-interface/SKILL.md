---
name: revit-performance-adviser-rule-interface
description: |
  实现自定义性能检查规则（IPerformanceAdviserRule）时使用。七方法：GetName、GetDescription、InitCheck、WillCheckElements、GetElementFilter、ExecuteElementCheck、FinalizeCheck。执行模型：InitCheck→GetElementFilter→ExecuteElementCheck→FinalizeCheck。Trigger：'实现性能规则/GetElementFilter'（performance rule interface）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.9（约p376-378）
tags: [performance-rule, interface, element-filter, lifecycle, validation]
related_skills: []
---

# IPerformanceAdviserRule 接口实现

## R — 原文 (Reading)

> 创建 IPerformanceAdviserRule 接口实例来创建 Performance Adviser 的新规则。这些规则可特定于图元，也可以是文件范围的规则。……
>
> — 宦国胜, 第5章 5.9 性能顾问规则接口（约p376-378）

---

## I — 方法论骨架 (Interpretation)

要实现一条自定义性能检查规则，就实现 `IPerformanceAdviserRule` 接口——它规定了七个方法，构成规则的生命周期：

1. **GetName() / GetDescription()**：规则的名字与说明（展示给用户/结果列表）。
2. **InitCheck()**：**文件级初始化**，对整份文件只跑一次——适合放文件级检查逻辑。
3. **WillCheckElements()**：返回 bool，告诉引擎"这条规则要不要进入图元级检查"——文件级规则可以返回 false 跳过后半程。
4. **GetElementFilter()**：返回一个元素过滤器——只有**通过过滤器的图元**才会进 ExecuteElementCheck。这是控制遍历开销的关键（如 `ElementClassFilter(typeof(Wall))` 只看墙）。
5. **ExecuteElementCheck()**：**图元级检查**，对每个被过滤出来的图元执行一次。
6. **FinalizeCheck()**：**收尾汇总**，把全部结果汇总并用 `FailureMessage` 报告问题（经事务 PostFailure 或由引擎收集）。

执行模型可以概括为：`InitCheck（文件级，一次）→ (WillCheckElements + GetElementFilter 决定去重/筛选) → ExecuteElementCheck（图元级，多次）→ FinalizeCheck（汇总）`。

性能要点：在百万图元的项目里，`GetElementFilter()` 是你的第一道闸门——只放行相关类型图元，`ExecuteElementCheck` 的调用次数就从"百万"降到"该类型数量"。文件级逻辑放 InitCheck、汇总放 FinalizeCheck，避免重复计算。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 只关心墙类图元的检查
- **问题**: V2 预测场景——自定义规则只关心墙类图元，如何在百万图元项目里控制检查开销？
- **方法论的使用**: WillCheckElements() 返回 true + GetElementFilter() 返回 ElementClassFilter(typeof(Wall))。
- **结论**: 只有通过过滤器的图元才进 ExecuteElementCheck。
- **结果**: 遍历开销大幅降低，检查聚焦。

### 案例 2: 文件级与图元级分工
- **问题**: 有些检查针对整份文件（如"项目存在空房间"），有些针对单个图元。
- **方法论的使用**: 文件级逻辑放 InitCheck（只跑一次）；结果汇总放 FinalizeCheck（经事务 PostFailure 报告）。
- **结论**: 三阶段生命周期 = InitCheck → (filter) → ExecuteElementCheck → FinalizeCheck。
- **结果**: 规则结构清晰，性能顾问特有的执行模型被正确利用。

### 案例 3: 结果经故障消息报告
- **问题**: 检查出的问题怎么报出去？
- **方法论的使用**: FinalizeCheck 用 FailureMessage 报告问题。
- **结论**: 复用故障消息机制。
- **结果**: 用户能看到统一的故障提示，可接入故障流水线自动处理。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写自定义性能检查规则（公司质量规范）。
2. 担心规则在百万图元项目里太慢，要加元素过滤器。
3. 不知道 InitCheck / ExecuteElementCheck / FinalizeCheck 各放什么逻辑。
4. 规则结果要作为故障消息报告。

### 语言信号 (用户的话里出现这些就应激活)

- "实现一条性能检查规则" / "implement a performance adviser rule"
- "IPerformanceAdviserRule 七方法" / "the seven methods of the rule interface"
- "只检查墙/门/房间这类图元" / "check only walls / doors / rooms"
- "GetElementFilter 怎么用？" / "use GetElementFilter to limit checks"
- "InitCheck 和 FinalizeCheck 放什么？" / "what goes in InitCheck and FinalizeCheck"

### 与相邻 skill 的区分

本 skill 与 `revit-performance-adviser-rules` 区分：那个管"注册/执行/呈现"（外壳），本 skill 管"规则内部怎么写"（内核）。与 `revit-failure-definition-registration` 区分：FinalizeCheck 报告故障时可能复用故障消息，但规则本身不是自定义故障。GetElementFilter 与通用元素过滤（ElementFilter）是同族思想、不同上下文。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **设计规则生命周期**：明确哪些逻辑放 InitCheck（文件级一次）、哪些放 ExecuteElementCheck（图元级）、哪些放 FinalizeCheck（汇总）。
   - 完成标准: 每段逻辑归位；明确是否需要图元级检查（否则 WillCheckElements 返回 false）。

2. **实现七方法**：
   - GetName/GetDescription：稳定、可读。
   - InitCheck：文件级检查/初始化。
   - WillCheckElements：决定是否进入图元级。
   - GetElementFilter：**返回精确过滤器**（如 ElementClassFilter），控制遍历范围。
   - ExecuteElementCheck：对每个命中图元执行检查逻辑。
   - FinalizeCheck：汇总结果，用 FailureMessage 报告。
   - 完成标准: 七方法齐全、类型签名正确、过滤器精确。

3. **注册并验证**：
   - 用 AddRule 注册（见 revit-performance-adviser-rules），在小项目上跑一遍：确认只检查目标类型、结果报告正确、无异常。
   - 在更大模型上实测耗时。
   - 完成标准: 检查范围精确、结果正确、性能可控。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 内置规则已覆盖检查项——不必自定义实现。
- 一次性分析不需要复用为规则——直接写分析代码更快。

### 作者在书中警告的失败模式

- **GetElementFilter 太宽**——过滤器不精确，ExecuteElementCheck 被调用百万次，性能灾难。
- **文件级逻辑误放到 ExecuteElementCheck**——每个图元重算一遍文件级结果，浪费。
- **FinalizeCheck 忘报告**——检查做了但结果没报出去，用户无感。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版规则接口的签名（如是否引入 context 参数）可能有调整，应按当前 SDK 实现。
- 未讨论"规则里做复杂几何计算"的成本——过滤器只能挡图元类型，挡不住单图元内部的高成本检查，需自己评估。

### 容易混淆的邻近方法论

- `GetElementFilter`（规则内嵌过滤器） vs 独立 ElementFilter 查询：一个是"让引擎只喂给我该看的图元"，一个是"我自己挑图元"——前者是性能顾问的执行约定。
- `InitCheck`（文件级一次） vs `ExecuteElementCheck`（图元级多次）：放错位置是常见 bug。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
