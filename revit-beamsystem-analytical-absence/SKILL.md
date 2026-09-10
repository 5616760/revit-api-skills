---
name: revit-beamsystem-analytical-absence
description: |
  反例 skill：在 BeamSystem（梁系统）自身上找 AnalyticalModel 会失败——聚合系统图元不直接暴露分析模型，分析能力下沉到成员（每根梁）。须先 GetBeamIds() 打散再逐梁查询。Trigger：BeamSystem 没有 AnalyticalModel、编译期找不到方法、"为什么查不到/是不是 API 缺失"、Grid 类似情况、聚合下沉设计原理。不适用于：梁系统/单体图元支撑信息的常规查询与直查-先分解分派表（GetBeamIds 打散的 happy path）——那是 revit-analytical-model-supports 的触发面；单根梁的支撑查询、分析位置读取。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.2.5（约p307-308）
tags: [revit-api, counter-example, beamsystem, analytical-model, aggregate-element]
related_skills:
  - slug: revit-analytical-model-supports
    relation: depends-on
---

# 反例：在 BeamSystem 自身上找 AnalyticalModel（不存在）

## R — 原文 (Reading)

> 绘制梁系统时，虽然可以选择墙作为支撑，但由于 BeamSystem 无 AnalyticalModel 属性，因此无法直接获取其支撑信息。解决方案是调用 GetBeamIds() 方法，来检索梁的 AnalyticalModelSupport 集合。
>
> — 宦国胜, 第4章 4.2.2.5（约p307–p308）

---

## I — 方法论骨架 (Interpretation)

一个揭示 Revit 聚合图元设计的反例：

1. **表面现象**：在 BeamSystem 上调 `.GetAnalyticalModel()`（或访问其分析模型相关成员）——成员不存在/返回不了，编译期找不到方法只是表象。
2. **深层规律**：**聚合系统图元把分析能力下沉到成员**——BeamSystem 自己没有 AnalyticalModel，但系统里每根梁有。分析信息按成员分发。
3. **正确姿势**：`beamSystem.GetBeamIds()` 取全部梁 Id → 逐根梁 `GetAnalyticalModelSupports()` 查支撑。
4. **可泛化的判断**：类似图元还有 Grid 等——遇到"系统族/聚合图元在分析层不直接暴露"，一律"先打散到 leaf 图元再查询"。
5. **连带限制**：`IsElementFullySupported()` 也不适用于 BeamSystem——通用预检也要在 leaf 层做。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 想直接读梁系统的支撑信息
- **问题**: 开发者在 BeamSystem 上找 AnalyticalModel/AnalyticalModelSupports，失败（绘制梁系统时虽可选墙做支撑，但系统本身无分析模型）。
- **方法论的使用**: 按"打散到 leaf"规律改写——`GetBeamIds()` 检索梁集合，再逐梁查 AnalyticalModelSupport 集合。
- **结论**: 编译期找不到方法不是 API 版本问题，而是设计如此。
- **结果**: 改为逐梁查询后，支撑信息完整取得；IsElementFullySupported 也逐梁应用。

### 案例 2: 推广到其他聚合图元
- **问题**: 哪些图元还会像 BeamSystem 一样"分析模型不直接暴露"？
- **方法论的使用**: 用规律预测——系统族/聚合图元（BeamSystem、Grid 等）常不直接暴露，遇未知图元先按"打散到成员"假设试探。
- **结论**: 写通用支撑分析工具时先按类型分派：BeamSystem→GetBeamIds 分解；墙/板→直接查 AnalyticalModelSupports。
- **结果**: 工具对聚合与单体图元都能工作。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 编译期在 BeamSystem 上找不到 AnalyticalModel 相关成员，怀疑引用或版本问题。
2. 写通用分析工具时需要决定"哪些图元直查、哪些先分解"。
3. 在梁系统上做 IsElementFullySupported 检查，结果异常或不可用。
4. 用户问 Grid 等其他聚合图元为什么也没有分析模型。

### 语言信号 (用户的话里出现这些就应激活)

- "BeamSystem 没有 AnalyticalModel" / "BeamSystem has no AnalyticalModel"
- "编译找不到方法" / "compiler cannot find method on BeamSystem"
- "GetBeamIds" / "拆梁系统 / 分解梁系统"
- "聚合图元怎么查分析信息" / "aggregate element analytical info"

### 与相邻 skill 的区分

- 与 `revit-analytical-model-supports` 的关系：该 skill 是含五方向的完整支撑查询决策表；本 skill 专门讲其中 BeamSystem 分支“无分析支撑/模型”的事实及打散规律。
- 与 `revit-analytical-model-geometry` 的区别：该 skill 读单图元分析几何；本 skill 讲哪些图元根本没有可读分析模型。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认图元是聚合型**
   - 判断目标是不是 BeamSystem（或 Grid 等系统族/聚合图元）。
   - 完成标准: 类型已确认；单体图元（墙/板/单梁）直接走直查路径（判停，转支撑决策 skill）。
2. **打散到成员**
   - `beamSystem.GetBeamIds()` 取成员梁集合。
   - 完成标准: 拿到全部成员 Id，不再对 BeamSystem 自身访问任何 AnalyticalModel 成员。
3. **在成员层查询**
   - 逐根梁做 `GetAnalyticalModelSupports()` / `IsElementFullySupported()` 等操作，结果按需聚合回系统级报告。
   - 完成标准: 每根成员图元的分析信息已取得，聚合结果与需求一致。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单根梁/墙/板的支撑查询——直查即可，无需打散。
- 分析位置几何读取——单体图元走三件套；本 skill 只处理"有没有分析模型"的问题。
- 试图给 BeamSystem "补"一个分析模型——API 不提供此能力。

### 作者在书中警告的失败模式

- 在 BeamSystem 自身找 AnalyticalModel——不存在，`IsElementFullySupported` 也不适用。
- 把"编译期找不到方法"当版本问题反复换 API——设计如此，唯一出路是 GetBeamIds 分解。
- 期待系统级汇总：聚合图元不会替成员汇报分析信息。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中梁系统与分析模型体系均有变化（分析模型重构后 BeamSystem 相关访问路径需重核）。
- 书中未给出"聚合图元清单"，泛化规律（Grid 等）是推断性的，遇到新图元仍需逐一验证。

### 容易混淆的邻近方法论

- 支撑查询决策表——本反例是其中 BeamSystem 分支的深化。
- 分析模型守卫原则——那是"能力探测"，本 skill 是"能力不存在"。

---

## 相关 skills

- **revit-analytical-model-supports**（分析模型支撑（AnalyticalModelSupport）按图元类型的访问决策 · depends-on）— 本 skill 是支撑查询决策表中 BeamSystem 分支的反例知识，依赖该 skill 的完整决策表。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
