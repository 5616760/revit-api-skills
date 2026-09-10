---
name: revit-analytical-model-exception
description: |
  反例 skill：拿到 AnalyticalModel 不判定 IsSinglePoint/IsSingleCurve 直接调 GetPoint/GetCurve，会抛 InapplicableDataException——运行时崩溃级，主线程未捕获时 Revit 弹"遇到问题需要关闭"，未保存修改可能丢失。Trigger：分析模型异常、GetPoint 崩溃、Revit 闪退、插件稳定性。不适用于：正确的守卫写法（用守卫原则 skill）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.2.1（约p286）
tags: [revit-api, counter-example, analytical-model, exception, crash]
related_skills:
  - slug: revit-analytical-model-geometry
    relation: depends-on
---

# 反例：不使用 IsSinglePoint/IsSingleCurve 直接调 GetPoint/GetCurve 会崩溃

## R — 原文 (Reading)

> 否则将引发 Autodesk.Revit.Exceptions.InapplicableDataException。IsSinglePoint() 方法可用于确定分析模型是否可用单个点来表示。
>
> — 宦国胜, 第4章 4.2.2.1（约p286）

---

## I — 方法论骨架 (Interpretation)

一个"反着看才知道严重"的失败案例：

1. **错误写法**：拿到 Foundation（或任何图元）的 AnalyticalModel 后直接调 `GetPoint()`（或对墙调 `GetCurve()`）——不做任何能力判定。
2. **后果链**：对不适配的模型调用 → 抛 `Autodesk.Revit.Exceptions.InapplicableDataException` → 这是**运行时崩溃级**异常，不是可忽略的警告：
   - 主线程未捕获时，Revit UI 弹"Revit 遇到问题需要关闭"对话框；
   - 当前文档未保存的修改可能丢失；
   - 异常抛出点可能在事务中间，事务上下文一并丢失。
3. **正确写法**（对照）：先 `IsSinglePoint()` / `IsSingleCurve()` 判定分支（代码 4-13 正是用 GetCurve 先取分析曲线的正确示例）。
4. **定位价值**：当用户报告"插件让 Revit 崩了/闪退"，且调用栈涉及分析模型，第一怀疑对象就是这种裸调用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 拿 Foundation 的分析模型直接调 GetPoint()
- **问题**: 开发者按直觉对基础的分析模型直接调用 `GetPoint()`，期望拿到点。
- **方法论的使用**: 以反例视角验证——GetPoint 对非单点模型抛 InapplicableDataException；GetCurve() 同理。
- **结论**: 异常不是"返回失败"而是崩溃信号；正确做法是先 IsSinglePoint()/IsSingleCurve() 判定分支。
- **结果**: 误判成本被量化——运行时崩溃级、可拖垮 Revit 会话、丢事务上下文；改用前置判定后问题消失。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件运行后 Revit 弹"遇到问题需要关闭"或直接闪退，追查原因。
2. 用户贴出带 `InapplicableDataException` 的报错求解释。
3. 代码评审中发现裸 GetPoint/GetCurve 调用需要定级风险。
4. 用户问"这个异常能不能 try/catch 掉继续跑"。

### 语言信号 (用户的话里出现这些就应激活)

- "InapplicableDataException" / "GetPoint 抛异常"
- "Revit 遇到问题需要关闭 / 闪退" / "Revit crashed / needs to close"
- "分析模型读取崩了" / "analytical model read crashes"
- "try/catch 包住行不行" / "can I just catch it"

### 与相邻 skill 的区分

- 与 `revit-analytical-model-geometry` 的关系：该 skill 是完整使用框架；本 skill 只聚焦“不判定直接调”的失败模式，用于崩溃定位与风险定级。
- 与 `revit-analytical-model-guard-clause` 的区别：该 skill 给正面守卫写法；本 skill 是同一方法的反面叙述。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位崩溃点**
   - 在用户的调用栈/代码中找裸 `GetPoint()` / `GetCurve()` 调用，确认没有前置 IsSinglePoint/IsSingleCurve。
   - 完成标准: 找到至少一个无守卫调用点，或确认崩溃另有原因（判停）。
   - 判停条件: 若调用全部有守卫，转向其他崩溃假设，结束本 skill。
2. **评估损害面**
   - 确认异常是否发生在事务中间、是否有未保存修改风险。
   - 完成标准: 向用户说明可能的损害（文档状态、事务上下文丢失）。
3. **替换为守卫写法**
   - 改为三分支封装（IsSinglePoint→GetPoint；IsSingleCurve→GetCurve；else GetCurves），并明确告知"try/catch 兜底不能恢复文档状态，防御必须前置"。
   - 完成标准: 裸调用清零，复测无异常。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已经有守卫的正常代码——不需要本反例。
- 其他类型的 Revit 异常（ArgumentException 等）——各有语义，别套用本结论。

### 作者在书中警告的失败模式

- 主线程未捕获此异常——Revit 弹"遇到问题需要关闭"，未保存修改可能丢失。
- 异常发生在事务中间——事务上下文一并丢失，模型状态不可控。
- 以为 try/catch 就安全——文档状态已受影响。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版分析模型重构后异常行为与类名有变化；"先探测后调用"的结论仍稳。
- 书中未讨论 Revit 异常与 .NET 标准异常的映射关系，排错时可结合异常层次图。

### 容易混淆的邻近方法论

- 守卫原则 skill（正写）——本 skill（反写）的镜像。
- 分析模型三件套框架——上下文。

---

## 相关 skills

- **revit-analytical-model-geometry**（分析模型（AnalyticalModel）三件套：GetPoint / GetCurve / GetCurves · depends-on）— 本 skill 讲分析模型调用崩溃反例，需理解该 skill 的三件套使用框架。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
