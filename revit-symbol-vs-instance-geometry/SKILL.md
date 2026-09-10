---
name: revit-symbol-vs-instance-geometry
description: |
  用 GetInstanceGeometry 的几何面创建基于面的族/尺寸标注失败时——副本几何无有效 Reference，须改用 GetSymbolGeometry()（唯一非副本重载）；要项目坐标用 GeometryInstance.Transform.OfPoint 手动变换。只做算量/分析时副本恰好合适。不适用于：只需体积计算、不需创建参照。
  Trigger："基于面的族创建失败"、"尺寸标不上"、"hosted face family fails"、"symbol vs instance geometry"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.2（约 p218–p220）
tags: [symbol-geometry, instance-geometry, reference-semantics, transform, revit-geometry]
related_skills:
  - slug: revit-reference-stable-handle
    relation: composes-with
---

# 符号几何与实例几何的选择原则（GetSymbolGeometry 唯一非副本 vs GetInstanceGeometry 副本）

## R — 原文 (Reading)

> GetSymbolGeometry() 返回以族坐标系统表示的几何图形，是返回实际几何对象而非副本的唯一重载方法。GetInstanceGeometry() 返回以实例所在项目坐标系统表示的几何形状，总是副本，适合作为输出或分析工具的实现，不适合作为创建其他图元的参照。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.2 节（约 p218–p219）

---

## I — 方法论骨架 (Interpretation)

族实例的几何有两个来源，区别不在"坐标系"，而在"是不是原件"。

GetSymbolGeometry() 返回族坐标系的几何，并且是唯一返回"实际几何对象而非副本"的重载——原件几何上的面、边带有有效 Reference，可以被后续创建操作（基于面的族、尺寸标注）当作参照。

GetInstanceGeometry() 返回项目坐标系的几何，但总是副本——副本方便分析、测量、输出，但它不携带有效 Reference，拿它去创建新图元必然失败。

选择原则一句话：要"创建参照"用 SymbolGeometry + 手动 Transform.OfPoint 变到项目坐标；只做"分析输出"用 InstanceGeometry，省去变换。先判断用途，再选入口。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 符号几何 + 手动变换（3.7.2 代码 3-53）
- **问题**: 要读取族实例的体信息并映射到项目坐标。
- **方法论的使用**: GetSymbolGeometry() 取体信息，再用 GeometryInstance.Transform.OfPoint 把构成点变换到项目坐标系。
- **结论**: 符号几何 + 显式变换 = 既拿到原件几何又拿到项目坐标。
- **结果**: 原 s3-f05 并入本单元，作为非副本几何的标准用法。

### 案例 2: 柱内部 Solid 求交分析（3.7.2）
- **问题**: 提取柱内钢筋相关几何做求交分析。
- **方法论的使用**: GetInstanceGeometry() 获取柱内部 Solid，副本用于纯计算。
- **结论**: 分析场景下副本几何完全够用。
- **结果**: 见 s3-c01 完整案例。

### 案例 3: 梁几何带 Transform 重载（3.7.5）
- **问题**: 展开嵌套几何需要正确变换。
- **方法论的使用**: GetInstanceGeometry(geomInst.Transform) 带 Transform 重载。
- **结论**: 副本几何配合 Transform 可覆盖绝大多数展开需求。
- **结果**: 与 s3b-f01 的实例递归合并验证。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 用提取的族实例几何面创建基于面的族，结果失败或参照无效。
2. 用提取的几何面给构件加尺寸标注，标注报错或选不中。
3. 只是做体积/表面积/质心等分析，纠结要不要自己变换坐标系。
4. 需要项目坐标下的几何，但当前只有族坐标几何。

### 语言信号 (用户的话里出现这些就应激活)

- "用提取的面创建基于面的族失败" / "creating a face-based family from extracted geometry fails"
- "尺寸标注选不中提取的面" / "dimension cannot pick the extracted face"
- "取族坐标几何怎么变换到项目坐标" / "transform symbol geometry to project coordinates"
- "GetSymbolGeometry 和 GetInstanceGeometry 有什么区别" / "symbol vs instance geometry difference"

### 与相邻 skill 的区分

- 与 `revit-reference-stable-handle`：取得带有效 Reference 的几何后，其稳定句柄用于后续创建与持久化，组合使用。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **判断用途**
   - 完成标准: 明确当前目标是"创建新图元（需要有效 Reference）"还是"纯分析/输出（副本即可）"。
   - 判停条件: 若只是算量分析，直接用 GetInstanceGeometry() 并结束，不再做变换。

2. **选择几何入口**
   - 完成标准: 需要创建参照时调用 GetSymbolGeometry()；否则调用 GetInstanceGeometry()。

3. **处理坐标系**
   - 完成标准: 用 GeometryInstance.Transform.OfPoint（或 OfVector）把族坐标点变换到项目坐标；若后续创建操作失败，回头核查是否误用了副本几何。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 处理的是非族实例（如系统族的墙、楼板）：它们没有 Symbol/Instance 几何之分，走 revit-element-geometry-extraction。
- 只需要几何做体积/重心分析：副本几何即可，无需为了"非副本"付出 SymbolGeometry + 手动变换的代价。

### 作者在书中警告的失败模式

- GetInstanceGeometry 返回副本，副本几何无有效 Reference，作为创建参照必然失败——这是最常见的"几何能看不能用"事故。
- 以为 GetSymbolGeometry 返回的就是项目坐标：它返回族坐标，必须手动 Transform.OfPoint。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014 / .NET 4.0：新版 API 中 GetInstanceGeometry(transform) 重载仍是展开嵌套几何的标准做法；"唯一非副本重载"的契约在后续版本中保持。

### 容易混淆的邻近方法论

- 符号几何 ≠ "简化几何"：符号是族坐标系的原始几何，不是 LOD 简化。
- 副本语义与"缓存副本"：GetInstanceGeometry 每次返回新副本，别拿它做跨调用句柄。

---

## 相关 skills

- **revit-reference-stable-handle**（composes-with）：取得带有效 Reference 的几何后，其稳定句柄用于后续创建与持久化，组合使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
