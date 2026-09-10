---
name: revit-analytical-link-hub
description: |
  原则 skill：AnalyticalLink.Create 的参数是 Hub（中心）的 ElementId 而非图元 Id——连接永远建在"中心"之间，必须先把图元解析到所属 Hub（GetHub）。Trigger：AnalyticalLink.Create、Hub、连接两根柱/梁、分析连接、找不到怎么连、GetHub。不适用于：支撑查询、物理几何连接、MEP 连接件（Connector 体系）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2.4、4.2.5（约p296、p313）
tags: [revit-api, analytical-link, hub, structure, principle]
related_skills:
  - slug: revit-analytical-model-geometry
    relation: depends-on
---

# AnalyticalLink 永远连"中心（Hub）"而非"图元"——必须先取 Hub 再连

## R — 原文 (Reading)

> 静态方法 AnalyticalLink.Create() 创建一个新的分析连接，而不是直接连接两个图元，连接在两个"中心"之间创建。Hub 类表示两个或更多 Autodesk Revit 图元之间的连接。
>
> — 宦国胜, 第4章 4.2.4、4.2.5（约p296、p313）

---

## I — 方法论骨架 (Interpretation)

Revit 结构分析连接的间接寻址模型：

1. **连接的对象不是图元**：`AnalyticalLink.Create(doc, linkTypeId, startHubId, endHubId)` 的参数是 **Hub 的 ElementId**。Hub 表示两个或更多图元之间的连接（节点），一根结构梁可被多个 Hub 共享。
2. **图元→Hub 解析是必经前置步骤**：先扫描所有 Hub（`OfClass(typeof(Hub))`），用柱/梁的分析模型 Id 匹配 connector 的 Owner.Id，找到所属 Hub 再传给 Create。
3. **典型场景**：两根柱与一根梁在同一节点相交，要在两根柱之间建分析连接——不能传柱 Id，要找它们共同关联的 Hub。
4. **GetHub 辅助**：书中代码 4-24 用 `GetHub()` 从分析模型 Id 解析出 Hub 的 ElementId。

核心思想：这是"连接中心而非连接图元"的间接寻址抽象——多构件交汇的节点先物化为 Hub，连接发生在 Hub 之间。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 在同一节点相交的两根柱之间建分析连接
- **问题**: 两根柱与一根梁在同一节点相交，要在这两根柱之间建 AnalyticalLink，直接传柱的 ElementId 不行。
- **方法论的使用**: 因为 Create 的参数是 Hub Id——先 `OfClass(typeof(Hub))` 扫描所有 Hub，用柱分析模型 Id 匹配 connector.Owner.Id 找到所属 Hub，再以两个 Hub Id 调用 Create。
- **结论**: 图元→Hub 的解析是绕不开的前置步骤。
- **结果**: 分析连接正确建立在两柱共享的节点上。

### 案例 2: 用 GetHub 从分析模型解析 Hub
- **问题**: 手里有分析模型的 Id，需要对应的 Hub。
- **方法论的使用**: 书中代码 4-24 用 `GetHub()` 从分析模型 ID 直接解析出 Hub 的 ElementId，避免全量扫描匹配。
- **结论**: 有直接解析通道时优先用，省去遍历。
- **结果**: 解析一步完成，连接创建代码简洁。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写分析模型修正/校核工具，需要在构件之间补建分析连接。
2. 用户发现 AnalyticalLink.Create 传图元 Id 编译不过或行为不对，问参数到底是什么。
3. 处理节点偏移/释放（pinned/adjust）后重建分析连接。
4. 需要理解"一根梁被多个 Hub 共享"的拓扑，排查连接建错节点。

### 语言信号 (用户的话里出现这些就应激活)

- "连接两根柱/梁" / "create AnalyticalLink between columns / beams"
- "Hub / 中心 / 节点" / "Hub element / connection hub"
- "AnalyticalLink.Create 参数" / "GetHub"
- "为什么不能直接连图元" / "link elements directly"

### 与相邻 skill 的区分

- 与 `revit-analytical-model-geometry` 的关系：本 skill 建 AnalyticalLink（先取 Hub 再连），依赖该 skill 对分析位置的读取框架。
- 与 `revit-mep-connector-system-topology` 的区别：MEP 连接件是物理/逻辑连接件体系，与分析连接（AnalyticalLink）完全不同域，避免混淆。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **收集并解析 Hub**
   - 优先 `GetHub()` 从分析模型 Id 解析；否则 `OfClass(typeof(Hub))` 全量扫描，用构件分析模型 Id 匹配 connector.Owner.Id。
   - 完成标准: 拿到参与连接的每个构件对应的 Hub ElementId。
   - 判停条件: 若某构件解析不到 Hub（节点未形成），先检查构件是否真正交汇/模型是否已更新。
2. **创建连接**
   - `AnalyticalLink.Create(doc, linkTypeId, startHubId, endHubId)`，必要时选/建 linkType。
   - 完成标准: 连接创建成功且落在预期节点之间。
3. **验证拓扑**
   - 读回连接的起止 Hub，确认与预期节点一致；注意一根梁可被多个 Hub 共享，验证要精确到 Hub。
   - 完成标准: 连接拓扑与设计意图一致，无误连。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 查"谁支撑谁"——支撑是 AnalyticalModelSupport 体系，与 Link/Hub 无关。
- MEP 管道连接——用 Connector/物理连接件，不是 Hub。
- 物理几何连接（join/剪切）——走几何 API。

### 作者在书中警告的失败模式

- 直接把图元 ElementId 传给 AnalyticalLink.Create——参数是 Hub Id，语义完全不同。
- 假设构件↔Hub 一一对应——一根梁可被多个 Hub 共享，匹配时须精确。
- 跳过图元→Hub 解析，凭坐标猜连接——间接寻址不可绕过。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版分析模型重构后 Hub/AnalyticalLink 体系被新的节点-构件模型替代，概念思想（节点间接寻址）保留但 API 需重新核对。
- 书中对 Hub 生命周期能力（谁创建/删除 Hub）着墨少。

### 容易混淆的邻近方法论

- AnalyticalModelSupport——承载关系 vs 连接关系。
- MEP Connector——同名"连接"概念但不同体系。

---

## 相关 skills

- **revit-analytical-model-geometry**（分析模型（AnalyticalModel）三件套：GetPoint / GetCurve / GetCurves · depends-on）— 建分析连接前需先取 Hub 并理解分析位置语义，本 skill 依赖该 skill 的分析模型三件套。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
