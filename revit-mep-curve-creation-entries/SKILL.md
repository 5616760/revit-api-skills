---
name: revit-mep-curve-creation-entries
description: |
  需要以编程方式创建管道/风管/软管时调用：按"两点/两连接件/一点一连接件"三入口选型——锚定几何（点）还是锚定拓扑（连接件）。Trigger：NewPipe/NewDuct、创建风管管道、create pipe/duct、连接件之间创建、MEPCurve Create。不适用于：连接件遍历与系统归属（拓扑链 skill）、管径修改（参数 skill）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.3.1.1（约p297-298）
tags: [revit-api, mep, pipe, duct, creation]
related_skills:
  - slug: revit-mep-connector-system-topology
    relation: composes-with
  - slug: revit-mep-diameter-param
    relation: composes-with
---

# MEP 管道/风管创建的三种入口与三种创建方法

## R — 原文 (Reading)

> 有三种方法来创建新的风管、软风管、管道和管件。它们可以在两个点之间、两个连接件之间、或者一个点和一个连接件之间创建。此外，在两个点之间创建这些 MEPCurves 类型之一时，可以使用其对应的静态方法 Create()。
>
> — 宦国胜, 第4章 4.3.1.1（约p297–p298）

---

## I — 方法论骨架 (Interpretation)

MEP 干管创建（NewPipe / NewDuct / NewFlexPipe）的入口按"起点/终点类型"组织成矩阵：

1. **两点入口**：`NewDuct(xyz1, xyz2, ductType)`（或静态 `Create()`）——纯几何定位，不与任何既有构件发生关系。
2. **两连接件入口**：`NewDuct(conn1, conn2, ductType)`——自动对齐且系统归属正确，适合"接在既有管路上"。
3. **一点一连接件入口**：`NewDuct(xyz, conn, ductType)`——从既有连接件接出到指定位置。

选型本质是**锚定几何（点）还是锚定拓扑（连接件）**的权衡：要系统一致性和自动对齐就用连接件锚定；要自由布管就用点锚定。

配套要点：
- 创建前须先取合法 ElementType（PipeType/DuctType）；
- 创建后改尺寸不能写 `pipe.Diameter`（只读），须写 `RBS_PIPE_DIAMETER_PARAM` 等内建参数（见管径参数 skill）。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 从风机末端连接件自动连出风管
- **问题**: 用户选中一个风管末端和一段既有风管的连接件，要自动连出风管。
- **方法论的使用**: 按入口矩阵选"两连接件入口"——`NewDuct(conn1, conn2, ductType)`（书中代码 4-28），自动对齐且系统归属正确；两点入口无连接件对齐语义，不适合。
- **结论**: 入口选择本质是锚定几何还是锚定拓扑的权衡。
- **结果**: 新风管两端精确接在两个连接件上，系统属性自动继承。

### 案例 2: 创建后调整直径
- **问题**: 创建的管道需要改直径。
- **方法论的使用**: 不能写 `pipe.Diameter`（只读），必须 `pipe.get_Parameter(RBS_PIPE_DIAMETER_PARAM).Set(0.5)`。
- **结论**: 直径写入收敛到内建参数 API。
- **结果**: 直径修改成功，管线按新尺寸显示。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写布管/布风管脚本（自动连接设备、批量拉支管）。
2. 用户问"用点创建还是用连接件创建有什么区别"。
3. 从既有管路的连接件延伸新管段，要求系统归属正确。
4. 需要创建软管/软风管（FlexPipe 等同类入口）。

### 语言信号 (用户的话里出现这些就应激活)

- "创建风管/管道" / "create pipe / duct, NewDuct / NewPipe"
- "两个连接件之间接管" / "create duct between connectors"
- "静态 Create 方法" / "MEPCurve.Create two points"
- "自动接管 / 自动连接设备" / "auto-route piping"

### 与相邻 skill 的区分

- 与 `revit-mep-connector-system-topology` 的关系：本 skill 把连接件当作创建锚点；该 skill 讲连接件的拓扑遍历与归属，一创建一导航。
- 与 `revit-mep-diameter-param` 的关系：本 skill 覆盖创建入口选型，创建后写尺寸需复用该 skill 的直径参数通道。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确定锚定物**
   - 明确起点/终点是几何点（XYZ）还是连接件（Connector），据此选入口：点-点 / 连接件-连接件 / 点-连接件。
   - 完成标准: 入口已选定；若两端都是连接件，优先用两连接件入口（对齐+系统归属）。
   - 判停条件: 若取不到合法 ElementType（PipeType/DuctType），先补类型获取逻辑。
2. **创建**
   - 事务内调用 `NewPipe` / `NewDuct` / `NewFlexPipe`（或两点场景的静态 `Create()`）。
   - 完成标准: 得到新 MEPCurve 实例，位置/系统符合预期。
3. **后处理尺寸（如需）**
   - 经内建参数修改尺寸：`RBS_PIPE_DIAMETER_PARAM` / `RBS_DUCT_DIAMETER_PARAM` / `RBS_RECT_DUCT_WIDTH_PARAM` / `RBS_RECT_DUCT_HEIGHT_PARAM`。
   - 完成标准: 尺寸参数写成功，不出现对 Diameter 属性赋值的代码。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只遍历/检查既有系统连接——那是连接件拓扑链 skill 的领域，不涉及创建。
- 创建管件（Fitting）——管件通常由连接逻辑自动生成，不是三入口矩阵直接覆盖的对象。
- 修改既有管的系统属性——走系统 API。

### 作者在书中警告的失败模式

- 用两点入口硬接既有管路——丢失系统归属与自动对齐，需要事后手工补系统。
- 试图写 `pipe.Diameter` 属性改尺寸——只读，编译期即报错。
- 创建前未取合法 ElementType——创建调用直接失败。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 MEPCurve 创建 API 签名（如 Pipe.Create 带系统类型参数）有变化，三入口思想仍适用但需核对。
- 布线偏好（RoutingPreferenceManager）在 2014 覆盖有限。

### 容易混淆的邻近方法论

- 连接件拓扑链——创建锚点的"连接件"正是那个体系中的对象。
- 管径参数原则——尺寸写入通道。

---

## 相关 skills

- **revit-mep-connector-system-topology**（MEP 系统中"连接件→系统→设备"的拓扑链 · composes-with）— 本 skill 用连接件做创建锚点，该 skill 讲连接件/系统/设备拓扑链，两者组合覆盖 MEP 创建与导航。
- **revit-mep-diameter-param**（Pipe.Diameter 属性只读，修改必须走 RBS_PIPE_DIAMETER_PARAM 内建参数 · composes-with）— 创建后改管径需走该 skill 的直径参数通道，二者组合成完整创建流程。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
