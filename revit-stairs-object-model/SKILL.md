---
name: revit-stairs-object-model
description: |
  需要以编程方式访问楼梯构件（梯段/平台/支撑/栏杆）或判断楼梯是"按构件"还是"按草图"时调用。Trigger：Stairs、stairs runs / landings / supports、按构件楼梯、梯段路径、GetStairsPath、GetFootprintBoundary、楼梯类型 CutMark。不适用于：在楼梯编辑会话内创建构件（走 StairsEditScope 流程）、栏杆创建（受作用域隔离限制）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.10（约p256-263）
tags: [revit-api, stairs, component-access, stairs-type, railing]
related_skills: []
---

# 楼梯构件对象模型与访问路径（Stairs→Runs/Landings/Supports）

## R — 原文 (Reading)

> Stairs 类表示 Revit 中的楼梯图元并包含以下属性信息：踏板、踢面、层数，以及楼梯高度、底部和顶部标高。Stairs 类的方法可用于获取楼梯平台构件、楼梯梯段构件和楼梯的支撑。…… "按构件"楼梯由梯段、平台和支撑组成。每项构件都可从 Stairs 类检索。
>
> — 宦国胜, 第3章 3.10 楼梯和栏杆扶手（约p256–p263）

---

## I — 方法论骨架 (Interpretation)

访问楼梯数据前必须先回答一个问题：这个楼梯是**按构件**还是**按草图**建的？`Stairs.IsByComponent(doc, id)` 给出答案，两种模式的访问路径完全不同。对按构件楼梯，按三类构件分通道访问：

1. **梯段（StairsRun）**：`GetStairsPath()` 返回投影到基面的路径曲线（梯段在平面上的走向）；`GetFootprintBoundary()` 返回底部轮廓曲线；`BeginsWithRiser/EndsWithRiser` 判断端部是否以踢面收头。
2. **平台（StairsLanding）**：有 `Thickness` 等自身属性。
3. **支撑（Supports）**：`GetStairsSupports()` 返回的是一般 `Element`——API 未发布支撑专用类，要取支撑类型只能查参数。
4. **类型对象**：经 `GetTypeId()` 取 `StairsType` / `StairsRunType` / `StairsLandingType`；剪切标记经 `STAIRSTYPE_CUTMARK_TYPE` 内建参数取 `CutMarkType`。
5. **栏杆**：另走 `GetAssociatedRailings()`。

核心思想："先判定模式、再选通道"——跳过判定直接套某条通道，是楼梯代码出错的主因。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 取"梯段投影路径"与"底部轮廓"
- **问题**: 需要梯段在地面上的投影路径和底部轮廓两条不同的几何信息。
- **方法论的使用**: 区分两个语义不同的 API——`StairsRun.GetStairsPath()` 给投影到基面的路径曲线，`GetFootprintBoundary()` 给边界轮廓曲线，端部收头用 `BeginsWithRiser/EndsWithRiser` 判定。
- **结论**: "路径"与"轮廓"是两个概念，各自有专用方法，不能混用。
- **结果**: 按语义选对方法后，两条几何信息一次取全。

### 案例 2: 支撑构件没有专用类时的处理
- **问题**: `GetStairsSupports()` 返回的是普通 Element，没有 StairsSupport 类可用。
- **方法论的使用**: 承认"API 未发布支撑专用类"这一事实，转向查支撑 Element 的参数来获取类型等信息。
- **结论**: Revit 的对象模型不总是完整覆盖每类构件；支撑是"降级为一般 Element"的例子。
- **结果**: 通过参数通道仍能拿到支撑类型等必要信息，功能不受阻。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写楼梯明细/检查工具，需要遍历每部楼梯的梯段数、平台厚度、支撑清单。
2. 做楼梯几何分析（净高、路径走向、踢面收头）时需要取梯段路径与轮廓。
3. 需要读取/修改楼梯类型信息（StairsType、剪切标记 CutMarkType）。
4. 用户问"为什么 GetStairsSupports 返回的不是专用类、怎么取支撑类型"。

### 语言信号 (用户的话里出现这些就应激活)

- "楼梯的梯段/平台/支撑怎么遍历" / "iterate stairs runs / landings / supports"
- "梯段投影路径 / 底部轮廓" / "stairs path / footprint boundary"
- "按构件还是按草图" / "IsByComponent / by component stairs"
- "楼梯剪切标记类型" / "CutMarkType / STAIRSTYPE_CUTMARK_TYPE"

### 与相邻 skill 的区分

- 与 `revit-stairs-edit-scope-isolation` 的区别：该 skill 讲楼梯创建侧的作用域隔离限制（反例）；本 skill 管已存在楼梯的只读对象模型与构件导航（Stairs→Runs/Landings/Supports），一写一读、边界清晰。
- 与栏杆相关的区别：本 skill 仅经 GetAssociatedRailings() 做只读关联访问，不涉及栏杆创建。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判定楼梯模式**
   - `Stairs.IsByComponent(doc, stairsId)` 判断按构件/按草图。
   - 完成标准: 模式已确定；按草图楼梯不适用本 skill 的三通道访问，应改走草图（Sketch）相关路径并告知用户。
   - 判停条件: 若为按草图楼梯，报告差异并结束。
2. **分通道检索构件**
   - 梯段 `GetStairsRuns()`、平台 `GetStairsLandings()`、支撑 `GetStairsSupports()`、栏杆 `GetAssociatedRailings()`。
   - 完成标准: 四类构件集合均已取得（可为空集合）。
3. **按需取几何与类型**
   - 梯段路径/轮廓: `GetStairsPath()` / `GetFootprintBoundary()`；类型: `GetTypeId()` → StairsType/StairsRunType/StairsLandingType；剪切标记: 读 `STAIRSTYPE_CUTMARK_TYPE` 参数；支撑类型: 走支撑 Element 的参数。
   - 完成标准: 所需几何/类型信息已解析出，且没有对支撑调用不存在的专用类 API。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 要**创建/修改**梯段或平台——必须进入 StairsEditScope 编辑会话，本 skill 只覆盖只读访问。
- 要创建栏杆——被明确排除在楼梯编辑会话之外，见栏杆隔离限制 skill。
- 按草图创建的老式楼梯——构件三通道不适用。

### 作者在书中警告的失败模式

- 不判定 IsByComponent 就按构件模式访问——按草图楼梯没有梯段/平台/支撑可取。
- 期待支撑有专用类——`GetStairsSupports()` 只返回一般 Element，硬找专用类型 API 会失败。
- 把 `GetStairsPath()`（投影路径）与 `GetFootprintBoundary()`（轮廓）混为一谈。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。按构件楼梯是 2013 引入的新功能，2014 API 覆盖尚不完整（支撑无专用类即为证据）；新版 Revit 对楼梯 API 有增补，需核对。
- 楼梯路径编辑、多层楼梯等新能力在本书时代不可用。

### 容易混淆的邻近方法论

- StairsEditScope 编辑会话（创建路径）与作用域隔离反例——本 skill 的访问对象正是那个流程的产物。
- Sketch 草图模型——按草图楼梯的底层机制，与按构件模式并列。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
