---
name: revit-ifc-exporter-registration
description: |
  需要用自定义实现接管 Revit 的 IFC（或 Navisworks/几何）导出过程时调用：实现 IExporterIFC 接口并在 OnStartup 中用 ExporterIFCRegistry.RegisterIFCExporter 注册。核心约束：应用程序级单例、先到先得、一个会话只允许一个，后注册者抛 IllegalOperationException；注册后 UI 与 Document.Export 的 IFC 导出都会走它。Trigger：custom IFC exporter、ExporterIFCRegistry、IExporterIFC、自定义导出、接管 IFC 导出。不适用于：用内置导出程序常规导出。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.16.2/5.16.3 自定义导出（约p422-423）
tags: [revit-api, ifc-export, exporter-registry, plugin, customization]
related_skills: []
---

# 自定义 IFC / Navisworks 导出器注册

## R — 原文 (Reading)

> 要创建自定义 IFC 导出程序，请实现 IExporterIFC 接口并用 ExporterIFCRegistry 类注册。注意：ExporterIFCRegistry 是一个应用程序级的单例模式，且一个 Revit 给定会话只可注册一个 IExporterIFC。
>
> — 宦国胜, 第5章 5.16.2 导出 IFC / 5.16.3 自定义导出（约p422-423）

---

## I — 方法论骨架 (Interpretation)

当内置导出程序不够用时，Revit 允许插件**接管导出过程**。模式是"实现接口 + 启动时注册"，但带一个硬约束：

1. **实现接口**：写一个类实现 `IExporterIFC`，在接口方法里完成自定义的 IFC 写出逻辑。
2. **选注册时机**：在 `OnStartup`（外接程序启动）中调用 `ExporterIFCRegistry.RegisterIFCExporter(exporter)`——注册必须发生在任何 IFC 导出之前。
3. **理解接管范围**：注册成功后，**无论用户从 UI 还是 API（Document.Export）触发 IFC 导出**，走的都是这个自定义实现。
4. **接受单例规则**：ExporterIFCRegistry 是应用程序级单例，"服务第一"（先到先得）——一个 Revit 会话只允许一个 IExporterIFC。第二个插件再注册会抛 `IllegalOperationException`。
5. **同族机制**：Navisworks 导出器同属"插件导出程序"路线；更底层的 `CustomExporter` 则通过 RenderNode 节点树（GeometryNode/MaterialNode/LightNode 等）回调逐节点渲染模型，可自定义写出任意格式。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 两个插件竞争 IFC 导出权
- **问题**: 两个插件都实现了 IExporterIFC 并在各自 OnStartup 中注册，担心冲突。
- **方法论的使用**: 按"服务第一"原则分析——第一个完成注册的插件获得本会话 IFC 导出权。
- **结论**: 之后无论 UI 还是 Document.Export 触发的 IFC 导出都用第一个插件；第二个插件注册时收到 IllegalOperationException。
- **结果**: 排查"我注册的导出器没生效"类问题时，优先检查加载顺序与是否已被抢先注册。

### 案例 2: 用 CustomExporter 遍历几何做自定义写出
- **问题**: 需要按自己的文件格式导出模型几何与材质。
- **方法论的使用**: 走 CustomExporter 的节点树回调——遍历 GeometryNode/MaterialNode/LightNode 等 RenderNode，在回调中写出数据。
- **结论**: 导出内容与过程完全由插件控制，不依赖内置格式选项。
- **结果**: 实现了内置导出不支持的自定义格式输出。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 公司要求 IFC 导出符合自家扩展或特定 schema，内置导出器不满足，要接管导出过程。
2. 插件注册自定义导出器时抛 IllegalOperationException，或发现导出走的是别人的实现。
3. 需要自研格式/轻量化格式的模型导出（CustomExporter 节点树路线）。
4. Navisworks 集成需求——确认其"仅插件导出程序"的实现方式。

### 语言信号 (用户的话里出现这些就应激活)

- "自定义/接管 IFC 导出" / "custom IFC exporter / override IFC export"
- "ExporterIFCRegistry / RegisterIFCExporter / IExporterIFC"
- "注册导出器报 IllegalOperationException"
- "CustomExporter / RenderNode 遍历几何导出"

### 与相邻 skill 的区分

本 skill 为独立方法论，与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **实现导出器接口**
   - 按目标路线实现 `IExporterIFC`（IFC 接管）或基于 `CustomExporter` 的节点回调写出（自研格式）。
   - 完成标准: 导出器类编译通过，导出逻辑有明确输入（doc/视图）与输出（文件）。

2. **在 OnStartup 中注册并处理冲突**
   - `ExporterIFCRegistry.RegisterIFCExporter(exporter)`；用 try/catch 包住 `IllegalOperationException`，提示"本会话已被其他插件注册"。
   - 完成标准: 注册成功，或冲突被明确捕获并向用户报告（而非崩溃）。
   - 判停条件: 若捕获到已注册冲突，说明本插件导出器本会话不会生效，停止注册重试，报告先到者信息。

3. **验证接管效果**
   - 注册后触发一次 IFC 导出（UI 或 Document.Export），确认产物出自自定义实现。
   - 完成标准: 导出产物特征（自定义 schema/内容）可验证来自本插件；卸载插件后恢复内置行为（如可测）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 常规格式导出能满足需求——直接用内置 Document.Export 重载，不要为导出而接管。
- 只想调导出选项/映射表——那是 ExportOptions 与映射表 skill 的领域，无需替换导出器。
- 期望"多导出器并存按需切换"——单例先到先得规则不允许，一个会话只有一个 IExporterIFC。

### 作者在书中警告的失败模式

- 第二个注册者抛 IllegalOperationException——多插件环境下必须有冲突处理，否则插件启动即崩。
- 在 OnStartup 之外的时机注册，或在 IFC 导出已发生之后才注册——接管不生效或行为未定义。
- 误以为 UI 导出与 API 导出走不同实现——注册后两者都被接管，测试要覆盖两条入口。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：新版 IFC 导出器（开源 IFC exporter 的演化）与 CustomExporter 节点类型都有扩展，接口签名与节点种类需按当前版本核对。
- 书中未讨论多插件生态下的注册顺序治理（加载顺序决定先到者），工程上需要约定。

### 容易混淆的邻近方法论

- `revit-document-export-formats`（内置导出决策）——先确认内置不够用，再走本 skill。
- 更新器注册类 skill（OnStartup 注册 + 运行时触发）——模式相似但职责完全不同，勿混用注册对象。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
