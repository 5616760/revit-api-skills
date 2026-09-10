---
name: revit-element-geometry-extraction
description: |
  从 Element 提取几何（Solid/Faces/Edges/Curve）算量时。取切割前原始几何用 FamilyInstance.GetOriginalGeometry()；GeometryInstance 带 Transform 递归展开；仅驱动线用 Element.Location。不适用：仅要 Reference（revit-reference-stable-handle）、光线投影（revit-reference-intersector-raycast）、遍历面边界（revit-face-edge-loop-traversal）。
  Trigger："提取几何/遍历面"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.1 / 3.7.5（约 p210–p211、p221–p222）
tags: [geometry-extraction, geometryelement, solid, family-instance, revit-geometry]
related_skills:
  - slug: revit-reference-intersector-raycast
    relation: contrasts-with
  - slug: revit-geometry-utility-classes
    relation: contrasts-with
  - slug: revit-geometry-options-visibility
    relation: composes-with
  - slug: revit-symbol-vs-instance-geometry
    relation: composes-with
---

# 从 Element 提取几何的标准流程（GetOriginalGeometry 与 GeometryInstance 递归）

## R — 原文 (Reading)

> 墙几何体是由表面和边缘组成的体。用 Wall 类 Geometry 属性检索 Geometry.Element 实例。迭代 Object 属性，在 Faces 和 Edges 属性中获取一个包含所有几何表面和边缘的几何体实例。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.1 节（约 p210–p211）

---

## I — 方法论骨架 (Interpretation)

从图元取几何是一条固定流水线，四个决策点串起来：第一步，构造 Geometry.Options（是否求 Reference、视图、详细程度），它决定"返回什么级别的几何"；第二步，选择入口——`Element.Geometry` 返回"当前状态"几何（已连接/已切割/已开洞），`FamilyInstance.GetOriginalGeometry(options)` 返回"出厂状态"原始几何，两者语义不同，选错直接得到不同结果；第三步，遍历 GeometryElement 里的 GeometryObject，按类型分派处理 Solid（再进 Faces/Edges）、Curve、Mesh；第四步，遇到 GeometryInstance（嵌套族/带变换的实例化几何）必须用 `GetInstanceGeometry(geomInst.Transform)` 带 Transform 递归展开，否则只拿到一个壳。

最后一条分支：如果只是要驱动曲线（梁轴线、墙体中心线之类），根本不必进入几何树——直接从 `FamilyInstance.Location` 取 LocationCurve。几何提取不是"遍历一切"，而是按需求选路径。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 墙几何提取（3.7.1）
- **问题**: 需要读取墙体的全部表面和边缘。
- **方法论的使用**: Geometry.Options（ComputeReferences + DetailLevel）→ Element.Geometry → 遍历 GeometryObject → 转 Solid → Faces/Edges。
- **结论**: 墙几何体由表面和边缘组成，体级对象承载全部边界信息。
- **结果**: 该路径成为后续一切"取体"案例的模板。

### 案例 2: 梁的净几何与实例递归（3.7.5）
- **问题**: 要拿梁被连接、切割前的净几何做钢构算量。
- **方法论的使用**: 不用 Element.Geometry（连接后几何），改用 FamilyInstance.GetOriginalGeometry()；遇 GeometryInstance 用 GetInstanceGeometry(geomInst.Transform) 递归展开；驱动曲线另走 FamilyInstance.Location 的 LocationCurve。
- **结论**: "当前 vs 原始"与"实例是否递归展开"是两条正交决策。
- **结果**: 同一流程在柱中钢筋长度计算（3.7.2 代码 3-56）第三次复用。

### 案例 3: MEP 族连接件拉伸面检索（第4章）
- **问题**: 遍历 MEP 族实例几何定位连接件。
- **方法论的使用**: Options.ComputeReferences=true + View=ActiveView，同一流水线遍历 Solid。
- **结论**: 流水线对类别无关，MEP 与土建走同一框架。
- **结果**: 见 s4-c04 的独立验证。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要遍历构件的 Solid/Faces/Edges 做体积、表面积或碰撞计算。
2. 需要"连接/切割/开洞之前"的原始几何（算量、净尺寸复核）。
3. 几何树里出现 GeometryInstance，拿不到真正的体/面。
4. 要取梁/墙的驱动线（轴线），却误入了几何树遍历。

### 语言信号 (用户的话里出现这些就应激活)

- "提取这个构件的几何体" / "extract geometry from this element"
- "遍历墙体的面/边" / "iterate wall faces and edges"
- "要梁连接切割前的净几何" / "get original geometry before joins"
- "GeometryInstance 怎么展开" / "how to expand geometry instance"
- "取梁的轴线/驱动线" / "get the location curve of the beam"

### 与相邻 skill 的区分

- 与 `revit-reference-intersector-raycast`：本 skill 遍历几何树取几何本身，后者用 ReferenceIntersector 光线投影做命中查询——取几何 vs 查命中，手段对立。
- 与 `revit-geometry-utility-classes`：本 skill 是通用遍历流程，后者提供现成工具类快捷路径——两条对立的取几何路线。
- 与 `revit-geometry-options-visibility`：本 skill 管流程主干，后者管 Options 参数导致的缺几何/精度故障，组合成完整几何提取链路。
- 与 `revit-symbol-vs-instance-geometry`：本 skill 负责从哪取、怎么遍历，后者负责符号/实例的副本语义选择，组合使用。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **构造 Geometry.Options**
   - 完成标准: 决定 ComputeReferences（需要创建参照才开 true）、DetailLevel、是否指定 View；能说出每个选项为何如此设置。
   - 判停条件: 若需求只是轴线/中心线/驱动线，跳到步骤 4 改用 Location。

2. **选择几何入口：Geometry vs GetOriginalGeometry**
   - 完成标准: 明确"当前状态几何"还是"原始几何"；原始几何只在目标是 FamilyInstance 时可用，否则回到 Geometry。

3. **遍历并做类型分派**
   - 完成标准: 对 GeometryElement 顶层逐个 GeometryObject 分派：Solid→Faces/Edges，Curve→直接收集，Mesh→网格处理；遇 GeometryInstance 调用 GetInstanceGeometry(instance.Transform) 并递归进入步骤 3。
   - 判停条件: 不再出现新的 GeometryInstance 即停止递归；若顶层混有 Solid 与 GeometryInstance 两种类型，两者都要处理，不能只挑一种。

4. **驱动线分支**
   - 完成标准: 用 Element.Location 取 LocationCurve 或 LocationPoint，得到轴线级几何。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只需要某个面的 Reference 用于创建（尺寸、基于面的族）：那是 revit-reference-stable-handle 与 revit-symbol-vs-instance-geometry 的职责。
- 只需要光线投影查找遮挡：走 revit-reference-intersector-raycast（revit-reference-intersector-raycast）。
- 只需要把面边界导出为闭合环：走 revit-face-edge-loop-traversal（revit-face-edge-loop-traversal）。

### 作者在书中警告的失败模式

- 用 Element.Geometry 取"连接后几何"却声称是原始几何，算量会错。
- 遇到 GeometryInstance 不递归展开，遍历结果缺体缺面。
- GetOriginalGeometry 对非 FamilyInstance 调用无效。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 / .NET 4.0：`GeometryInstance.GetInstanceGeometry` 在现代版本仍可用，但部分场景（如修改后的族）语义需复核；`GetSymbolGeometry`/`GetInstanceGeometry` 的"副本 vs 非副本"契约在不同版本间保持不变。

### 容易混淆的邻近方法论

- `Element.Geometry` 与 `GetOriginalGeometry`：前者当前几何、后者原始几何，同名似而不同义。
- 几何树遍历与 Location 取线：一个是体量级几何，一个是驱动线级几何，用途不重叠。

---

## 相关 skills

- **revit-reference-intersector-raycast**（contrasts-with）：本 skill 遍历几何树取几何本身，后者用 ReferenceIntersector 光线投影做命中查询——取几何 vs 查命中，手段对立。
- **revit-geometry-utility-classes**（contrasts-with）：本 skill 是通用遍历流程，后者提供现成工具类快捷路径——两条对立的取几何路线。
- **revit-geometry-options-visibility**（composes-with）：本 skill 管流程主干，后者管 Options 参数导致的缺几何/精度故障，组合成完整几何提取链路。
- **revit-symbol-vs-instance-geometry**（composes-with）：本 skill 负责从哪取、怎么遍历，后者负责符号/实例的副本语义选择，组合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
