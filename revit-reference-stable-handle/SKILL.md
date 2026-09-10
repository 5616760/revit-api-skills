---
name: revit-reference-stable-handle
description: |
  需要长期持有几何句柄（跨会话恢复拾取的面、序列化存盘再还原）时。Reference 以灵活方式标识"几何表示树路径"，按 Pick 指针分四类参照（曲线端点/曲线/面/边），不可混用（线边界条件要线参照、面要面参照）。ConvertToStableRepresentation() 序列化为字符串，ParseFromStableRepresentation() 还原相同 Reference——ElementId（会话内整数）做不到。不适用于：单次会话内使用（ElementId 即可）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.7.3（约 p215–p216）
tags: [reference, stable-representation, geometry-handle, serialization, persistence]
related_skills: []
---

# Reference（参照）作为 Revit 几何 API 中的稳定句柄

## R — 原文 (Reading)

> 参照以灵活方式识别几何表示树路径，对创建图元有用。API 公开四类参照。Reference.ConvertToStableRepresentation() 可用于以字串存储几何参照，用 ParseFromStableRepresentation() 获取相同的 Reference。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.7.3 节几何变换与参照（约 p215–p216）

---

## I — 方法论骨架 (Interpretation)

在 Revit 里，"指着某个几何对象"不是存一个数字，而是一个 Reference——它是对"几何表示树路径"的灵活标识：树里每一层（图元、实例、面、边、端点）都可被引用。API 按 UI 拾取指针的类型公开四种参照：曲线端点、整条曲线、面、边。四类不可混用：创建线边界条件要线参照，创建面边界条件要面参照，拿错类型结果不对。

Reference 最值钱的能力是"稳定表示"：ConvertToStableRepresentation() 把它压成字符串，可存数据库、可写文件；下次会话 ParseFromStableRepresentation() 还原出完全相同的 Reference，仍指向同一几何对象。ElementId 只是会话内的整数，重启即失效；Reference 的稳定字符串是跨会话、跨用户仍然有效的几何句柄。

一句话模型：需要"现在用一下"→ ElementId；需要"记住这个几何，以后还能找回"→ Reference + 稳定表示。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 镜像墙的参照面（2.5）
- **问题**: 镜像墙需要定位镜像平面的参照。
- **方法论的使用**: HostObjectUtils.GetSideFaces 返回墙侧面 Reference 用于定位。
- **结论**: 创建/变换操作大量依赖 Reference 指向具体面。
- **结果**: 见 s2-c01。

### 案例 2: 尺寸创建的 References（3.6）
- **问题**: 给两个构件的面加尺寸。
- **方法论的使用**: Dimension.References 承载被标注的对象引用。
- **结论**: 尺寸的本质是"参照 + 参照"之间的距离关系。
- **结果**: 见 revit-dimension-identification。

### 案例 3: 边界条件的参照类型（第4章）
- **问题**: 创建不同维度的边界条件。
- **方法论的使用**: 点/线/面参照分别对应 NewPointBoundaryConditions / NewLineBoundaryConditions / NewAreaBoundaryConditions。
- **结论**: 参照类型与创建 API 一一对应，不可混用。
- **结果**: 链接图元交互同样依赖 Reference 转换（见 revit-link-reference-conversion）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件重启后要恢复用户上次拾取的几何面，继续未完成的任务。
2. 需要把"选中了哪个面"存进配置/数据库，下次打开还能精确还原。
3. 创建尺寸、边界条件、基于面的图元时需要正确的参照类型。
4. 处理链接文件的图元引用转换。

### 语言信号 (用户的话里出现这些就应激活)

- "记住用户选的这个面，重启后还能用" / "persist the picked face across sessions"
- "把 Reference 存成字符串" / "save a reference as a stable string"
- "创建尺寸/边界条件需要什么参照" / "what reference is needed for dimensions or boundary conditions"
- "stable representation" / "ConvertToStableRepresentation"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **判断引用类型**
   - 完成标准: 明确需要的是曲线端点/整条曲线/面/边中哪一类参照；对应调用 Curve.EndPointReference / Curve.Reference / Face.Reference / Edge.Reference。

2. **按用途使用**
   - 完成标准: 创建尺寸/边界条件时传入正确类型参照；创建基于面的图元时传面参照；类型不匹配则报错前先自查。

3. **需要跨会话时做稳定化**
   - 完成标准: 持久化前调用 ConvertToStableRepresentation() 存字符串；恢复时 ParseFromStableRepresentation() 还原并校验非 null、指向原对象。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单次会话内的临时引用：用 ElementId 更简单，Reference 是重武器。
- 不需要几何级引用的场景（如只改参数值）：按 Element 操作即可。

### 作者在书中警告的失败模式

- 四类参照不可混用：拿线参照建面边界条件、拿面参照建线边界条件都会失败或产生错误结果。
- ElementId 是会话内整数，重启即失效——跨会话必须走稳定字符串。
- 稳定表示依赖几何树的稳定性：几何被重建后，旧的稳定字符串可能解析失败，需兜底处理。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：稳定表示机制在新版本中保持；但链接文档、设计选项等复杂上下文下的解析结果与 2014 版略有差异，生产代码应做解析失败的兜底。

### 容易混淆的邻近方法论

- Reference 与 ElementId：一个指向"几何树路径"，一个指向"图元本体"，层级不同，不可互换。
- ConvertToStableRepresentation 与 ToString：前者是契约化的可还原表示，后者只是调试输出。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
