---
name: revit-data-storage-paths
description: |
  当需要在 Revit 模型图元上存储自定义数据且面临共享参数 vs 可扩展存储选型时调用。
  不适用于：已有明确存储方案、仅需读写已有参数值。
  关键 trigger 信号："存数据到图元 store data on element"、"共享参数 shared parameter"、"可扩展存储 extensible storage"、"隐藏数据 hidden data"、"明细表导出 schedule export"。
  核心决策轴：数据消费者是用户/UI/明细表 → 共享参数；数据消费者是第三方程序且需结构化/隐藏 → 可扩展存储。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.1 导言 (约p315)
tags: [data-storage, shared-parameter, extensible-storage, schema, revit-api]
related_skills:
  - slug: revit-binding-type-vs-instance
    relation: depends-on
  - slug: revit-extensible-storage-pipeline
    relation: depends-on
---

# 共享参数与可扩展存储两种路径

## R — 原文 (Reading)

> Revit API 提供了两种在模型中存储数据的方法。第一种是共享参数，若定义为可见则在属性窗口中用户可见。另一种是可扩展存储，允许创建自定义数据结构，对用户总是不可见的，但通过第三方程序按读/写访问模式访问。
>
> — 宦国胜, 第5章 5.1 导言 (约p315)

---

## I — 方法论骨架 (Interpretation)

在 Revit 模型图元上存储自定义数据有两条路径，选型本质是"数据消费者是谁"：

**共享参数**：
- 键值标签式存储，字段类型有限（int/double/string/ElementId 等）
- 可定义 visible 属性，visible 时在属性窗口中用户可见
- 可绑定到图元类别（但不是全部），可出现在明细表、tag、ODBC 导出中
- 通过共享参数文件（.txt）管理定义，GUID 唯一标识

**可扩展存储**：
- 结构化数据存储，支持 Array/Map/Entity 嵌套字段
- 对 Revit UI 完全不可见，不会出现在属性窗口或明细表中
- 通过 Schema/Entity 机制，支持读写访问权限控制（AccessLevel）
- 第三方程序通过 API 按权限读写

决策轴：若数据需要出现在明细表/tag/属性窗口/ODBC 导出中 → 必须共享参数；若数据是程序内部状态、需隐藏或需结构化嵌套 → 必须可扩展存储。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 管道标注工具的数据存储选型

- **问题**: 需要把"上次检查日期+检查人ID+历史数组"存到每根管道上，且用户不应在属性窗口看到
- **方法论的使用**: 按数据消费者决策——消费者是第三方程序（非用户/UI），且需要 Array 嵌套字段 → 选可扩展存储
- **结论**: 共享参数只能做键值标签、字段类型受限、visible 后会在属性窗口出现，无法满足需求
- **结果**: 用 ExtensibleStorage Schema 存储结构化检查记录，对用户完全隐藏

### 案例 2: 明细表可导出的参数存储

- **问题**: 需要存储一个"成本"参数，要求出现在明细表中并可导出
- **方法论的使用**: 数据消费者是明细表/导出 → 必须共享参数，可扩展存储对 UI 和导出完全不可见
- **结论**: 选共享参数，绑定到 Walls 类别
- **结果**: 成本参数出现在墙明细表中，可 ODBC 导出

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 开发插件需要在图元上存自定义数据，不确定用共享参数还是可扩展存储
2. 需要存储结构化/嵌套数据（数组、Map），共享参数无法满足
3. 需要数据对用户隐藏（不在属性窗口出现）
4. 需要数据出现在明细表/tag/ODBC 导出中

### 语言信号 (用户的话里出现这些就应激活)

- "存数据到图元 / store data on element / 自定义数据存储"
- "共享参数 vs 可扩展存储 / shared parameter vs extensible storage"
- "隐藏数据 / hidden data / 不在属性窗口显示"
- "明细表导出 / schedule export / ODBC"

### 与相邻 skill 的区分

- 与 `revit-binding-type-vs-instance` 的关系：本 skill 是共享参数 vs 可扩展存储的两条路径选型；该 skill 是共享参数内部的绑定模式（Type/Instance）选型。
- 与 `revit-extensible-storage-pipeline` 的关系：本 skill 做选型决策；选定可扩展存储后，实现细节走该 skill 的六步流水线。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **识别数据消费者与数据结构需求**
   - 消费者：用户/UI/明细表/导出 → 共享参数；第三方程序 → 可扩展存储
   - 结构：键值 → 共享参数；嵌套/Array/Map → 可扩展存储
   - 可见性：需可见 → 共享参数（visible=true）；需隐藏 → 可扩展存储或共享参数（visible=false）
   - 完成标准: 明确消费者、结构复杂度、可见性需求

2. **按选型路径执行**
   - 共享参数 → 定义参数文件 → 绑定到类别 → 读写参数值
   - 可扩展存储 → 建 Schema → 创建 Entity → 赋值 → 挂载到图元
   - 完成标准: 数据成功存储到目标图元
   - 判停条件: 若需要明细表导出则必须共享参数，跳到共享参数路径

3. **验证数据可被目标消费者访问**
   - 共享参数：在属性窗口/明细表中确认可见
   - 可扩展存储：通过 API 读取确认数据完整
   - 完成标准: 数据被正确的消费者按预期访问

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已有明确存储方案且无需选型——直接执行对应路径
- 临时计算数据不需要持久化——用内存变量即可

### 作者在书中警告的失败模式

- 共享参数不能绑定到全部图元类别——部分类别不支持参数绑定
- 可扩展存储数据对 UI 和导出完全不可见——选错会导致数据无法在明细表中使用

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，可扩展存储是较新 API，后续版本可能放宽字段类型限制或新增访问模式

### 容易混淆的邻近方法论

- 共享参数 visible 属性 vs 可扩展存储的"对 UI 不可见"——前者可控可见性，后者天然不可见
- 共享参数的 GUID 唯一标识 vs 可扩展存储的 Schema GUID——两者都有 GUID 但用途不同

---

## 相关 skills

- **revit-binding-type-vs-instance**（共享参数绑定（Binding）的两种模式：TypeBinding vs InstanceBinding · depends-on）— 共享参数存储路径的选型依赖该 skill 的绑定模式（Type/Instance）知识。
- **revit-extensible-storage-pipeline**（可扩展存储（Extensible Storage）六步流水线 · depends-on）— 可扩展存储分支的实现细节依赖该 skill 的六步流水线。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
