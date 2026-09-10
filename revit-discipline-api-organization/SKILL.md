---
name: revit-discipline-api-organization
description: |
  在规划跨规程（Architecture/Structure/MEP）通用插件时调用：用"图元类/分析层/拓扑/规程设置"四维度预估每个规程的 API 能力边界。Trigger：结构还是 MEP、哪些图元有哪些 API、通用插件支持多规程、discipline namespace、能力边界预估。不适用于：单一规程内的具体 API 用法（走对应规程 skill）。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第4章 4.2/4.3 开篇（约p292、p297）
tags: [revit-api, discipline, mep, structure, api-map]
related_skills:
  - slug: revit-element-retrieval-four-entries
    relation: depends-on
  - slug: revit-analytical-model-geometry
    relation: composes-with
  - slug: revit-mep-connector-system-topology
    relation: composes-with
  - slug: revit-mep-curve-creation-entries
    relation: composes-with
---

# Revit 各规程特有 API 的整体组织逻辑（规程=领域接口层，四维度切分）

## R — 原文 (Reading)

> Revit API 的 Revit MEP 部分，提供对 Revit 模型中暖通空调（HVAC）和管道数据的读取和写入访问。包括：遍历系统中的风管、管道、管件、连接件；添加、删除和更改风管、管道及其他设备；获取和设置系统属性；确定系统是否连接良好；访问机械设置；管理布线配置。
>
> — 宦国胜, 第4章 4.2 / 4.3 开篇（约p292、p297）

---

## I — 方法论骨架 (Interpretation)

Revit 平台 API 之下，Architecture / Structure / MEP 三大规程各有独立命名空间与类层次，但共享统一的 Element 基类模型。判断"某类图元在 API 里能做什么"，用四个维度切分：

1. **图元类维度**：每个规程有独有图元 API——Structure 有 Truss/Rebar/AnalyticalModel 系；MEP 有 Pipe/Duct/Connector 系。
2. **分析层维度**：只有 Structure 有 AnalyticalModel / AnalyticalLink——MEP 图元**永远没有**分析模型，所以管道不能做结构分析连接。
3. **拓扑维度**：只有 MEP 有 MEPSystem / Connector / LogicalConnection 的系统归属——结构和建筑图元没有"系统"概念。
4. **规程设置维度**：MEP 有 PipeSettings / DuctSettings / RoutingPreferenceManager；Structure 有自己的分析设置。

核心思想：编码前先用四维度预判——知道哪一维存在哪类 API，就能在写代码前预估能力边界，避免"以为有、实际没有"的返工。底层 Element 模型统一保证跨规程的通用操作（过滤、参数、事务）可以共用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 通用插件要同时支持结构和 MEP 项目
- **问题**: 一个通用插件要覆盖结构和 MEP 项目，如何预估每类图元能做什么、不能做什么？
- **方法论的使用**: 按四维度切分预测——结构有 Truss/Rebar、MEP 有 Pipe/Duct/Connector（图元维）；只有 Structure 有分析模型（分析维）；只有 MEP 有系统拓扑（拓扑维）；MEP 有 PipeSettings 等规程设置（设置维）。
- **结论**: "管道做结构分析连接"这类需求在 API 层面就不成立，应在方案期排除。
- **结果**: 编码前的能力边界表直接决定了插件架构（通用层 + 规程扩展层）。

### 案例 2: 用底层统一 Element 模型写通用逻辑
- **问题**: 跨规程的通用功能（过滤、参数读写、事务）是否要每规程写一套？
- **方法论的使用**: 附录 B FAQ 的多规程参数/图元问题印证底层统一 Element 模型——通用逻辑基于 Element 基类一套即可。
- **结论**: 规程差异体现在领域层，底座是共享的。
- **结果**: 通用层写一次，规程层只补各维度的特有 API。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 规划一个新插件的 API 选型，想知道目标功能在某规程里"有没有 API 可用"。
2. 用户问"结构图元能不能像管道那样查系统归属"或"管道能不能做分析连接"这类跨维问题。
3. 学习 Revit API 时需要一个整体地图来安放各规程的类。
4. 排查"编译期找不到某类"——如 BeamSystem 上找不到 AnalyticalModel，需判断是版本问题还是维度缺失。

### 语言信号 (用户的话里出现这些就应激活)

- "结构还是 MEP 的 API / 这个功能哪个规程有" / "Structure vs MEP namespace / discipline API"
- "通用插件支持三大规程" / "plugin supporting all disciplines"
- "管道有分析模型吗" / "do pipes have analytical model"
- "MEP 能做什么" / "MEP API capabilities"

### 与相邻 skill 的区分

- 与 `revit-element-retrieval-four-entries` 的关系：本 skill 是领域间的组织地图，底层依赖通用图元检索底座，二者是上层地图与底层底座的关系。
- 与 `revit-analytical-model-geometry`、`revit-mep-connector-system-topology`、`revit-mep-curve-creation-entries` 的关系：这三个是具体规程维度的展开，本 skill 负责选型与预判，不重复具体 API 方法论。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **明确目标图元与所需能力**
   - 列出要操作的图元类别与要执行的能力（几何/分析/拓扑/设置）。
   - 完成标准: 得到"图元 × 能力"的需求清单。
2. **按四维度预判能力边界**
   - 逐项判断该能力落在哪个维度、该规程是否有此维度的 API（如 MEP 无分析模型、Structure 无系统拓扑）。
   - 完成标准: 每项需求标记为"有直接 API / 需变通 / 无 API 不可行"。
   - 判停条件: 若发现核心需求落在缺失维度（如管道分析连接），立即向用户报告不可行并结束选型。
3. **分层设计**
   - 通用逻辑（过滤/参数/事务）基于 Element 底座写一次；规程特有逻辑按维度挂接各自命名空间。
   - 完成标准: 架构中每条需求都有明确的 API 落点或已声明替代方案。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已确定规程、要写具体实现——直接进入对应规程的 skill（分析模型、MEP 创建、系统拓扑等）。
- 纯几何/视图/文档等平台层问题——与规程维度无关。

### 作者在书中警告的失败模式

- 把"每个规程有自己的命名空间"当作全部知识，忽略维度差异——以为能像结构那样给 MEP 图元建 AnalyticalLink，方案期就错了。
- 在缺失维度上硬凑（用几何模拟拓扑、用参数模拟分析层），成本远超收益。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中分析模型体系已重构（AnalyticalToPhysicalAssociation 等）、MEP 有更多系统 API，四维度框架本身仍适用但成员需更新。
- 2014 的"分析模型"概念与后来的分析-物理分离模型不同，直接套旧类名会失配。

### 容易混淆的邻近方法论

- 分析模型三件套 / MEP 拓扑链——都是本地图中的具体节点。
- Element 检索四入口——跨规程通用的底座方法。

---

## 相关 skills

- **revit-element-retrieval-four-entries**（图元检索四法决策框架 · depends-on）— 本 skill 是规程维度间的组织地图，底层检索能力依赖通用图元检索四法。
- **revit-analytical-model-geometry**（分析模型（AnalyticalModel）三件套：GetPoint / GetCurve / GetCurves · composes-with）— 分析模型三件套是本 skill 地图中“分析规程”维度的具体展开。
- **revit-mep-connector-system-topology**（MEP 系统中"连接件→系统→设备"的拓扑链 · composes-with）— MEP 拓扑链是本 skill 地图中“MEP 规程”维度的展开之一。
- **revit-mep-curve-creation-entries**（MEP 管道/风管创建的三种入口与三种创建方法 · composes-with）— MEP 创建入口是本 skill 地图中“MEP 规程”维度的创建侧展开。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
