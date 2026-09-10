---
name: revit-performance-adviser-rules
description: |
  注册/执行性能检查规则时使用。PerformanceAdviser 兼规则注册中心+执行引擎。何时调用：插件提供性能检查；跑全部或子集规则；报告模型性能问题。何时不调用：只读建议不落地。Trigger：'性能检查/PerformanceAdviser/AddRule/ExecuteRules'（performance adviser, execute rules, execute all rules）。规则生命周期：AddRule 在 OnStartup、DeleteRule 在 OnShutdown，否则残留注册。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.9（约p375-376）
tags: [performance-adviser, rules, registration, execution, validation]
related_skills:
  - slug: revit-performance-adviser-rule-interface
    relation: composes-with
  - slug: revit-failure-definition-registration
    relation: contrasts-with
---

# PerformanceAdviser 规则注册与执行

## R — 原文 (Reading)

> PerformanceAdviser 用来添加或删除检查用的规则、启用和禁用规则……在应用程序启动期间使用 AddRule() 创建新规则，在应用程序关闭期间使用 DeleteRule() 注销它。……
>
> — 宦国胜, 第5章 5.9 性能顾问（约p375-376）

---

## I — 方法论骨架 (Interpretation)

`PerformanceAdviser`（性能顾问）是 Revit 内置的**性能检查框架**，它同时扮演两个角色：

1. **规则注册中心**：负责添加/删除规则、启用/禁用规则。规则就是一段"检查模型有没有性能问题"的逻辑（如"门装反了""房间太小"）。
2. **执行引擎**：负责运行规则并收集结果。

关键 API 与约束：

- **注册与注销**：`AddRule()` 在**应用启动期间**（OnStartup）创建规则；`DeleteRule()` 在**应用关闭期间**（OnShutdown）注销。生命周期必须与插件对齐——否则关闭时残留注册。
- **执行方式**：
  - `ExecuteAllRules(documents)`：执行给定文件列表里的**所有**规则。
  - `ExecuteRules(document, ruleIdList)`：只执行**选定的**规则子集——想跳过内置规则、只跑自己的规则，就用这个。
- **结果呈现**：规则执行后返回**故障消息列表**（FailureMessage），直接接入 Revit 故障处理体系展示给用户。

方法论要点：先注册（OnStartup 一次性），后执行（按需全量或子集），最后把结果当故障消息交给用户/故障流水线。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 只跑两条自定义规则
- **问题**: V2 预测场景——想只跑"翻转的门"和"过小房间"两条性能规则、跳过其他内置规则。
- **方法论的使用**: ExecuteRules(document, ruleIdList) 精确指定规则子集。
- **结论**: ExecuteAllRules 跑全部；ExecuteRules 跑子集。
- **结果**: 检查开销受控，只报告关心的性能问题。

### 案例 2: 规则生命周期管理
- **问题**: 插件规则什么时候注册、什么时候注销？
- **方法论的使用**: AddRule 在 OnStartup、DeleteRule 在 OnShutdown。
- **结论**: 规则生命周期与插件对齐。
- **结果**: 关闭时无残留注册，不污染后续会话。

### 案例 3: 结果接入故障体系
- **问题**: 检查出的性能问题怎么展示？
- **方法论的使用**: 执行返回故障消息列表。
- **结论**: 可直接接入故障处理体系展示（弹窗/自动处理均可）。
- **结果**: 性能问题与常规故障统一呈现，体验一致。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件要实现自定义性能检查（按公司标准查模型质量问题）。
2. 需要跑"部分规则子集"而不是全部内置规则。
3. 编写/排查规则的注册注销时机（OnStartup/OnShutdown）。
4. 想把性能检查结果作为故障消息呈现给用户。

### 语言信号 (用户的话里出现这些就应激活)

- "自定义性能检查规则" / "custom performance check rules"
- "PerformanceAdviser 怎么注册规则？" / "how to add a performance adviser rule"
- "只想跑几条规则 / ExecuteRules" / "run a subset of rules"
- "AddRule / DeleteRule 什么时候调？" / "when to call AddRule and DeleteRule"
- "性能问题怎么报给用户？" / "report performance issues as failures"

### 与相邻 skill 的区分

本 skill 与 `revit-performance-adviser-rule-interface` 配合：前者管规则注册/执行/呈现（框架外壳），后者管规则内部实现（七方法契约）。与 `revit-failure-definition-registration` 区分：性能规则结果虽复用故障消息机制，但规则的产生与注册走 PerformanceAdviser，与自定义故障注册不同源。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定要注册的规则**：列出规则清单（每条的用途、目标文件范围、是否图元级/文件级）。
   - 完成标准: 规则清单明确，且明确要跑"全部"还是"子集"。

2. **注册与注销**：
   - OnStartup：对每条规则调用 AddRule()。
   - OnShutdown：对每条规则调用 DeleteRule()。
   - 完成标准: 注册/注销成对；关闭后无残留。

3. **执行并呈现结果**：
   - 全量：ExecuteAllRules(documents)；子集：ExecuteRules(document, ruleIdList)。
   - 把返回的故障消息列表接入故障处理体系/UI。
   - 完成标准: 检查执行完成，结果已呈现，无规则注册泄漏。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只做一次性分析、不需要注册成可复用规则——直接写分析代码即可。
- 用内置规则就够——不必自定义。

### 作者在书中警告的失败模式

- **AddRule/DeleteRule 生命周期不齐**——OnStartup 注册、OnShutdown 必须注销，否则残留。
- **ExecuteAllRules 可能跑掉你不想要的规则**——需要精确控制时用 ExecuteRules 子集。
- **结果故障消息处理不当**——性能问题也会走故障流水线，别让它变成弹窗刷屏。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版 PerformanceAdviser 的注册方式（如支持在 app 级还是文档级）与规则状态（启停）有演进，应按当前 SDK 核对。
- 未讨论"规则在大型项目上的执行耗时"——性能顾问本身也要控制性能开销。

### 容易混淆的邻近方法论

- `ExecuteAllRules` vs `ExecuteRules`：全量 vs 子集——不是同义词。
- 性能顾问（检查规则） vs 故障定义（业务故障）：前者产生"建议"，后者产生"错误/警告"，复用消息管道但语义不同。

---

## 相关 skills

- revit-performance-adviser-rule-interface（composes-with）：本 skill 管注册/执行/呈现（框架外壳），规则内部实现见该 skill。
- revit-failure-definition-registration（contrasts-with）：性能规则结果复用故障消息机制，但规则注册走 PerformanceAdviser，与自定义故障注册不同源。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
