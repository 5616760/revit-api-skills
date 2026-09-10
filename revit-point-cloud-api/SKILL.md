---
name: revit-point-cloud-api
description: |
  在 Revit 里创建/访问/显示点云时使用。两条路径：格式受支持走客户端 API（PointCloudType.Create→PointCloudInstance.Create 两步）；自有格式走引擎 API（实现 IPointCloudEngine 注册解析）。何时调用：导入扫描点云、要读点做分析（GetPoints）。何时不调用：格式受支持只需展示。Trigger：'导入点云/自定义格式/IPointCloudEngine/GetPoints'（point cloud, point cloud engine, custom format）。先判格式，再选路径。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.10（约p379-381）
tags: [point-cloud, engine, client-api, scanning, custom-format]
related_skills:
  - slug: revit-point-cloud-filter
    relation: composes-with
---

# 点云创建与访问框架（客户端 API vs 引擎 API）

## R — 原文 (Reading)

> Revit API 提供了两种使用点云的方法。第一种方法允许创建新的点云实例……第二种方法允许使用自己的云引擎并处理非支持的文件格式……要在 Revit 文件中新建点云，先创建 PointCloudType，再用它创建 PointCloudInstance。……
>
> — 宦国胜, 第5章 5.10 点云（约p379-381）

---

## I — 方法论骨架 (Interpretation)

Revit 的点云支持分**两条路径**，选择依据是"你的格式受不受 Revit 原生支持"：

**路径 A：客户端 API（格式受支持）**
两步创建：
1. `PointCloudType.Create(...)` 创建点云**类型**——需要指定**引擎标识符**（引擎决定怎么解析格式）。
2. 用该类型 `PointCloudInstance.Create(...)` 创建点云**实例**放进模型。

访问点数据：`GetPoints(filter, averageDistance, numPoints)` 批量取点，可用 PointCloudFilter 限制搜索范围（见 revit-point-cloud-filter）；遍历有迭代器与不安全指针（IntPtr）两种方式。

**路径 B：引擎 API（自有格式）**
如果客户有自己格式（如私有 .xyz 变体），Revit 不认——就实现 `IPointCloudEngine` 接口，通过 `PointCloudEngineRegistry` 注册自定义引擎，让 Revit 通过标准的 `IPointCloudAccess` / `IPointSetIterator` 访问你的数据。这样自定义格式也能在模型里显示、支持选取。

方法论要点：**先判格式，再选路径**——受支持走 A（快），不受支持走 B（实现引擎）。无论哪条路，点云创建都是"类型 + 实例"两步。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 客户自有格式的点云
- **问题**: V2 预测场景——客户有自己的 .xyz 点云格式，Revit 不认识，想直接在模型里显示并支持选取。
- **方法论的使用**: 走引擎 API——实现 IPointCloudEngine 并注册，用自定义引擎解析自有格式。
- **结论**: 内置引擎路径只认 Revit 支持格式；自定义格式必须走引擎路径。
- **结果**: Revit 通过标准 IPointCloudAccess/IPointSetIterator 访问自有格式数据。

### 案例 2: 受支持格式的创建流程
- **问题**: 格式受支持时怎么建点云？
- **方法论的使用**: 客户端 API——先 PointCloudType.Create（需引擎标识符），再 PointCloudInstance.Create。
- **结论**: 类型+实例两步。
- **结果**: 点云进入模型，可显示、可选取。

### 案例 3: 点云数据访问
- **问题**: 需要读取点云的点做分析（如扫描偏差）。
- **方法论的使用**: GetPoints(filter, averageDistance, numPoints) + 迭代器/不安全指针遍历。
- **结论**: 用过滤器限制搜索量（revit-point-cloud-filter），批量取点。
- **结果**: 大数据量点云的高效访问。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 要导入激光扫描/无人机点云到 Revit 模型。
2. 客户有私有点云格式，需要在 Revit 里显示与选取。
3. 要读取点云数据做偏差分析/体积计算。
4. 在"客户端 API"与"引擎 API"之间做架构选择。

### 语言信号 (用户的话里出现这些就应激活)

- "怎么导入点云？" / "import a point cloud into Revit"
- "PointCloudType / PointCloudInstance 创建" / "create point cloud type and instance"
- "自定义点云格式怎么支持？" / "support a custom point cloud format"
- "IPointCloudEngine 引擎注册" / "implement and register a point cloud engine"
- "读取点云的点做分析" / "read points with GetPoints"

### 与相邻 skill 的区分

本 skill 与 `revit-point-cloud-filter` 配合：本 skill 讲创建/访问/引擎选择，过滤是访问时的空间限制工具（GetPoints 的 filter 参数）。与几何提取类 skill 区分：点云是离散点集而非构件几何，访问模型不同，勿套用图元几何的读法。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断格式支持**：确认点云格式是否在 Revit 支持的引擎格式内（.rcp/.rcs 等）。
   - 受支持 → 走客户端 API（步骤 2A）。
   - 不支持 → 走引擎 API（步骤 2B）。
   - 完成标准: 路径已选定且有依据。

2A. **客户端 API**：
   - `PointCloudType.Create(doc, engineIdentifier, ...)` 建类型。
   - `PointCloudInstance.Create(doc, typeId, transform, ...)` 建实例。
   - 完成标准: 类型与实例创建成功，点云在模型中可见。

2B. **引擎 API**：
   - 实现 `IPointCloudEngine`（解析自有格式，暴露 IPointCloudAccess/IPointSetIterator）。
   - 用 `PointCloudEngineRegistry` 注册。
   - 走客户端 API 引用该引擎创建类型/实例。
   - 完成标准: 引擎注册成功，自定义格式点云可显示、可选取。

3. **访问与验证**：
   - 用 `GetPoints(filter, avgDistance, numPoints)` 取点，验证数据正确。
   - 检查显示、选取、搜索都正常。
   - 完成标准: 点云全链路可用；大数据量下访问性能可接受。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 点云格式受支持且只需简单导入展示——引擎 API 是过度设计。
- 不需要读取/选取点云——纯显示场景可走更轻的路径。

### 作者在书中警告的失败模式

- **格式不受支持却走客户端 API**——创建失败或数据解析错误。
- **PointCloudType.Create 需要引擎标识符**——漏引擎标识符无法创建。
- **大数据量点云直接全量取点**——不配合过滤器（revit-point-cloud-filter），性能灾难。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版点云引擎接口与支持格式清单（如 E57、las 的支持演进）变化较大，应以当前 SDK 与格式列表为准。
- 未讨论点云在视图中的渲染性能（Octree 结构、显示抽稀）细节——大数据量工程实践需要补充。

### 容易混淆的邻近方法论

- "点云类型" vs "点云实例"：前者是数据/格式的定义（含引擎标识符），后者是放进模型的一次摆放（含 transform）——两步别混。
- 客户端 API vs 引擎 API：不是"两个平行方案随便选"，而是"格式决定走哪条"。

---

## 相关 skills

- revit-point-cloud-filter（composes-with）：GetPoints 的 filter 参数即由该 skill 构造，访问点云时配套使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
