---
name: revit-mep-connector-system-topology
description: |
  需要遍历 MEP 系统（从设备经 ConnectorManager→Connector→AllRefs 递归下游）、创建系统（NewPipingSystem/NewMechanicalSystem）或判断"连接良好"时调用。Trigger：MEPSystem、Connector、AllRefs、遍历系统、connected well、连接件归属。不适用于：管道/风管创建入口选型、尺寸参数修改。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.3.1.4 + 4.3.2（约p302、p304）
tags: [revit-api, mep, connector, mepsystem, topology]
related_skills:
  - slug: revit-analytical-link-hub
    relation: contrasts-with
---

# MEP 系统中"连接件→系统→设备"的拓扑链

## R — 原文 (Reading)

> 新建系统所需的最后一点信息是 NewPipingSystem() 的 PipeSystemType 或是 NewMechanicalSystem() 的 DuctSystemType。新风管和新管道可以在两个连接件之间创建。连接件用以连接主体与风管、管道及电气设备，它们由连接件的 Domain 属性获得。
>
> — 宦国胜, 第4章 4.3.1.4 + 4.3.2（约p302、p304）

---

## I — 方法论骨架 (Interpretation)

MEP 的世界是一张三层拓扑图，导航链固定：

1. **完整导航链**：`Element（设备）` → `MEPModel` → `ConnectorManager` → `Connectors`（集合） → 每个 `Connector` → `MEPSystem` / `AllRefs`。
2. **连接件属性体系**：`Connector.Domain`（域：管道/风管/电气）、`Direction`（流向进出）、`MEPSystem`（系统归属）、`AllRefs`（引用集）、`ConnectorType`（物理/逻辑）。
3. **系统创建**：`NewPipingSystem(baseEquipmentConnector, connectorSet, PipeSystemType)` / `NewMechanicalSystem(..., DuctSystemType)`——基础设备的连接件 + 成员连接件集 + 系统类型三要素。
4. **物理 vs 逻辑**：物理连接件可见（真实接口），逻辑连接件不可见——`AllRefs` 只返回**物理**连接件。
5. **"连接良好"的判定**：基于物理拓扑（从设备递归遍历 AllRefs 下游）；系统成员等逻辑归属要靠 `MEPSystem` 引用判断，不能靠连接件遍历。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 从风机+末端创建机械系统
- **问题**: 把风机与末端设备组织成一个供风系统。
- **方法论的使用**: 书中代码 4-29——用基础设备连接件 + 成员连接件集 + DuctSystemType 调 `NewMechanicalSystem()`。
- **结论**: 系统创建的三要素（基础连接件/成员集/系统类型）缺一不可。
- **结果**: 机械系统建立，成员图元归入系统，可整体查属性。

### 案例 2: 判断连接件的归属与连接对象
- **问题**: 检查某个连接件接在什么上、属于哪个系统。
- **方法论的使用**: 书中代码 4-30——用 `Connector.MEPSystem`（系统归属）、`IsConnected`（是否已连接）、`AllRefs`（引用的连接对象）组合判断。
- **结论**: 三属性组合覆盖"归属/状态/对端"三个查询维度。
- **结果**: 连接件的状态检查代码可复用于任何 MEP 图元。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 检查一个供气/供水系统是否"连接良好"（有没有断点、漏接）。
2. 需要从基础设备出发遍历整个系统做统计/校核。
3. 程序化创建系统（把散管归入系统）。
4. 用户发现 AllRefs 查不到某些"连接"，疑惑逻辑连接件去向。

### 语言信号 (用户的话里出现这些就应激活)

- "遍历系统 / 系统连接良好" / "iterate MEPSystem / connected well"
- "ConnectorManager / AllRefs / Connector"
- "创建机械/管道系统" / "NewMechanicalSystem / NewPipingSystem"
- "物理连接件和逻辑连接件" / "physical vs logical connector"

### 与相邻 skill 的区分

- 与 `revit-analytical-link-hub` 的区别：本 skill 讲 MEP 连接件的拓扑链（连接件→系统→设备）；该 skill 讲分析模型的 AnalyticalLink，两者是完全不同的连接域，避免误用。
- 与 `revit-mep-curve-creation-entries` 的区别：该 skill 用连接件做创建锚点；本 skill 把连接件当拓扑对象做遍历与归属。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位导航链入口**
   - 设备/管件图元 → `MEPModel.ConnectorManager.Connectors` 取连接件集合。
   - 完成标准: 拿到待分析对象的连接件集合（可为空）。
   - 判停条件: 若图元无 MEPModel（非 MEP 设备），本链不适用，报告后结束。
2. **读连接件属性**
   - 按 Domain/Direction/MEPSystem/AllRefs/ConnectorType 组合判定每个连接件的归属、状态与对端。
   - 完成标准: 每个连接件的五元组信息明确；明确 AllRefs 只含物理连接件。
3. **递归遍历或建系统**
   - 遍历：从基础设备沿 AllRefs 递归下游（物理拓扑）；建系统：`NewPipingSystem` / `NewMechanicalSystem`（基础连接件+成员集+系统类型）。
   - 完成标准: 全系统图元已覆盖（连通性结论可信）或系统创建成功且成员归属正确；逻辑归属经 MEPSystem 引用核实。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只想创建一段管/风管——走创建入口 skill，不必动拓扑。
- 结构分析连接——那是 AnalyticalLink/Hub 体系，与 Connector 无关。
- 电气系统的详细电路分析——连接件 Domain 覆盖电气但本书重点在 HVAC/管道。

### 作者在书中警告的失败模式

- 用 AllRefs 期待看到逻辑连接——只返回物理连接件，逻辑连接件对应用层不可见。
- 用连接件遍历判断系统成员关系——逻辑归属要看 MEPSystem，两条通道混用得出错结论。
- 建系统时漏三要素之一（基础连接件/成员集/系统类型）——创建失败或系统不完整。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 MEP 系统 API（如系统类型管理、网络分析）有扩展，导航链结构基本延续。
- 2014 的 Connector 属性体系对后续版本新增的布线偏好、连接件坐标系细节覆盖不足。

### 容易混淆的邻近方法论

- 创建入口矩阵——连接件在那里是"锚点"，在这里是"拓扑节点"。
- 规程四维度地图——本 skill 是其拓扑维度展开。

---

## 相关 skills

- **revit-analytical-link-hub**（AnalyticalLink 永远连"中心（Hub）"而非"图元"——必须先取 Hub 再连 · contrasts-with）— 该 skill 是分析连接（AnalyticalLink），本 skill 是 MEP 连接件/系统/设备拓扑，两者是完全不同的连接域。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
