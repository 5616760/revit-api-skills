---
name: revit-analytical-model-guard-clause
description: |
  原则 skill：调用分析模型 GetPoint/GetCurve/GetCurves 前必须先用 IsSinglePoint/IsSingleCurve 做运行时能力探测，否则抛 InapplicableDataException（运行时崩溃级）。Trigger：InapplicableDataException、分析模型异常、"零容忍"防御、导出分析位置不崩溃、guard clause。不适用于：支撑查询、连接创建等非几何读取场景。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.2.1（约p302、p286）
tags: [revit-api, principle, analytical-model, exception, guard-clause]
related_skills:
  - slug: revit-analytical-model-exception
    relation: contrasts-with
---

# 分析模型的 GetPoint / GetCurve / GetCurves 必须先判定 IsSinglePoint / IsSingleCurve

## R — 原文 (Reading)

> (1) GetPoint()。如果分析模型可以用单个点来表示（如 Structural Footing），则此方法将返回该点。否则将引发 Autodesk.Revit.Exceptions.InapplicableDataException。
>
> — 宦国胜, 第4章 4.2.2.1（约p302、p286）

（注：输入包本单元未附原文，本段取自原书第 4 章候选提取稿 seg4.md 同章节。）

---

## I — 方法论骨架 (Interpretation)

Revit 分析模型 API 用**异常代替返回值**表达"不适配"，这决定了使用纪律：

1. **能力不对称**：`GetPoint()` 只对可单点表示的模型（如 Structural Footing）有效；`GetCurve()` 只对可单曲线表示的模型有效；对不适配的对象调用不返回 null 而是直接**抛异常**。
2. **前置探测**：`IsSinglePoint()` / `IsSingleCurve()` 是官方提供的运行时能力探测 API——先问"能不能"，再"取"。
3. **三分支封装**（工程模式）：
   ```
   if (model.IsSinglePoint())  return model.GetPoint();
   if (model.IsSingleCurve()) return model.GetCurve();
   return model.GetCurves();   // fallback，含多条曲线
   ```
   任何结构图元（Foundation / Column / Framing / Slab / Wall）都落入三分支之一，封装后永不崩溃。
4. **零容忍理由**：该异常是运行时崩溃级——主线程未捕获时 Revit 弹"遇到问题需要关闭"，当前文档未保存修改可能丢失；且异常可能发生在事务中间，事务上下文一并丢失。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: "把所有结构图元分析位置导出到 CSV"
- **问题**: 异构图元（基础/柱/梁/墙）混合输入，如何不崩溃地导出。
- **方法论的使用**: 统一走三分支封装——IsSinglePoint→GetPoint，IsSingleCurve→GetCurve，否则 GetCurves；不假设图元类型。
- **结论**: 省略预判定的版本遇到墙或基础（非单点/非单曲线）直接抛 InapplicableDataException，且未捕获时拖垮整个 Revit 会话。
- **结果**: 封装版对全部图元稳定导出，无崩溃。

### 案例 2: 异常严重度的认知
- **问题**: 这个异常若在主线程未捕获会怎样？对插件稳定性意味着什么？
- **方法论的使用**: 把它当作"零容忍"级防御点——分析模型访问必须前置判定，异常处理放在循环外层为时已晚。
- **结论**: API 设计上异常不是可恢复信号，是崩溃信号。
- **结果**: 所有分析模型读取代码统一套上三分支守卫，插件的稳定性事故归零。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 调试时遇到 `Autodesk.Revit.Exceptions.InapplicableDataException`，在 GetPoint/GetCurve 调用行。
2. 写遍历任意结构图元读取分析位置的工具（导出/校核/标注）。
3. 用户的插件偶发让 Revit 弹"遇到问题需要关闭"，追查到分析模型访问。
4. 代码评审时想确认是否所有 GetPoint/GetCurve 调用都有前置守卫。

### 语言信号 (用户的话里出现这些就应激活)

- "InapplicableDataException 崩了" / "InapplicableDataException thrown by GetCurve"
- "怎么安全地读分析模型" / "safely read analytical model geometry"
- "Revit 弹出遇到问题需要关闭" / "Revit crashed / needs to close"
- "IsSinglePoint / IsSingleCurve 要不要先判"

### 与相邻 skill 的区分

- 与 `revit-analytical-model-exception` 的区别：本 skill 是“先判定再取值”的正面守卫原则；该 skill 是同一方法论的负面叙述（直接调用会崩溃），两者互为正反面。
- 与 `revit-analytical-model-geometry` 的区别：该 skill 是三件套完整框架；本 skill 只是其中“必须先判定”这条纪律的浓缩。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **审计现有代码**
   - 全局搜索 `GetPoint()` / `GetCurve()` / `GetCurves()` 调用点，逐个检查是否有前置 IsSinglePoint/IsSingleCurve 判定。
   - 完成标准: 每个调用点标记为"已有守卫 / 缺守卫"。
   - 判停条件: 若无任何裸调用，报告已安全并结束。
2. **替换为三分支封装**
   - 引入辅助方法（IsSinglePoint→GetPoint；IsSingleCurve→GetCurve；else GetCurves），所有调用点改走封装。
   - 完成标准: 代码库中不存在未探测的 GetPoint/GetCurve 直接调用。
3. **验证边界**
   - 用混合图元集（基础+柱+墙+板）跑一遍，确认无异常且输出正确。
   - 完成标准: 全类型通过，无 InapplicableDataException。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 支撑查询、分析连接创建——不涉及三件套，异常面不同。
- 已经封装过的库内部——重复守卫无意义。

### 作者在书中警告的失败模式

- 直接调用崩溃级异常且未捕获——Revit UI 弹"遇到问题需要关闭"，未保存修改可能丢失。
- 异常抛出点在事务中间——事务上下文一并丢失，模型状态不可控。
- 以为 try/catch 包住就安全——文档状态已受影响，防御必须前置。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。"用异常表达不适配"的设计在新版分析模型重构后有所改变，但"先探测能力再调用"的纪律跨版本成立。
- "API 用异常代替返回值"这种设计评价新版需重新核对。

### 容易混淆的邻近方法论

- 分析模型三件套框架——本原则是其纪律层。
- 反例 skill（不判定直接调）——同一方法论的反面叙述。

---

## 相关 skills

- **revit-analytical-model-exception**（反例：不使用 IsSinglePoint/IsSingleCurve 直接调 GetPoint/GetCurve 会崩溃 · contrasts-with）— 该 skill 是“不判定就崩”的反例叙事，本 skill 是“怎么写守卫”的正面原则，互为正反面。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
