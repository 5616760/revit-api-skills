---
name: revit-internal-units-conversion
description: |
  当 API 读出的数值'不对劲'（AsDouble 返回英尺）、或用户输入毫米要写入模型时调用。Revit 内部：长度英尺、角度弧度，其余基本量公制。取值 AsDouble=内部裸值；展示用 Conv
  ertFromInternalUnits 或 AsValueString；输入用 ConvertToInternalUnits 或 SetValueString。铁律：显示与存储间永远做显式单位转换。
  不适用：整数/无单位参数。trigger：单位转换、英尺还是毫米、UnitUtils、internal units。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p064-065
tags: [units, unitutils, internal-units, numeric-conversion]
related_skills:
  - slug: revit-parameter-storage-types
    relation: composes-with
  - slug: revit-indexed-property-get-set
    relation: composes-with
---

# Revit 内部单位体系与 UnitUtils 转换

## R — 原文 (Reading)

> Revit 有七个基本量，每个都有其自己的内部单位：长度—英尺；角度—弧度；质量—千克；时间—秒；电流—安培；温度—开尔文；照度—坎德拉。
> 由于 Revit 以英尺存储长度，而其他基本量采用公制单位，因此涉及长度的导出单位均为非标准单位。
>
> — 宦国胜, 第1章 1.4.5 / 表 1-13（约 p064–p065）

---

## I — 方法论骨架 (Interpretation)

Revit API 的数值世界和用户看到的界面世界**不是同一套单位**：

- **内部单位（API 里的裸值）**：长度一律 **英尺**、角度一律**弧度**；质量千克、时间秒、电流安培、温度开尔文、照度坎德拉。这是七套内部单位，混着用。
- **显示单位（用户在界面上看到的）**：由项目单位设置决定（毫米、米、英尺-英寸……），UI 负责格式化。

因为长度用英制、其他基本量用公制，凡是"长度参与运算的导出单位"（如力 = kg·ft/s²）都是非标准组合——**千万别手推换算公式**。

因此 API 读写的正确姿势分三条路：

1. `parameter.AsDouble()` → 拿到**内部单位裸值**（英尺），不能直接给用户看。
2. 展示给用户 → `UnitUtils.ConvertFromInternalUnits(value, DisplayUnitType)`，或用 `parameter.AsValueString()` 直接拿项目单位格式化好的字符串（如 "200"）。
3. 接收用户输入 → `ConvertToInternalUnits(value, DisplayUnitType)` / `SetValueString` 转回内部单位再写模型。

铁律：**显示和存储之间永远隔着一层显式单位转换**，裸值跨过转换直接拼 UI 是经典 bug。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 墙长度参数读取（第 2 章 2.3.5）
- **问题**: `AsDouble()` 读墙的 Length 返回什么？
- **方法论的使用**: 认识到返回的是英尺内部值；用 `AsValueString()` 拿 "200" 字符串或 `AsDouble()` 拿 -20 双精度裸值。
- **结论**: 两条取值路径分别对应用户可读与程序内部两种语义。
- **结果**: 代码正确区分了显示值字符串与内部单位数值（代码 2-27 / 图 2-4）。

### 案例 2: 墙/楼板创建的高度值（seg2b）
- **问题**: 创建墙时高度参数填什么单位？
- **方法论的使用**: 填内部单位（英尺），用户输入值先 `ConvertToInternalUnits`。
- **结论**: 所有 Create 类 API 的尺寸参数都要求内部单位。
- **结果**: 创建代码与用户期望尺寸一致。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 读出来墙长是 13.1234，用户说"应该是 4000mm"——怀疑单位错。
2. 用户输入"3000"（毫米）想写入参数，直接塞 Set 发现模型里是 3000 英尺。
3. 写几何创建代码（墙/楼板/管线）不知道参数填什么单位。
4. 用 AsDouble 拼 UI 显示，数字总是不对。

### 语言信号 (用户的话里出现这些就应激活)

- "AsDouble 返回什么单位 / 怎么是英尺"
- "单位转换 / UnitUtils"
- "用户输入毫米怎么转"
- "AsValueString 和 AsDouble 区别"
- "internal units / feet / convert to mm / unit conversion / ConvertFromInternalUnits"

### 与相邻 skill 的区分

- 与 `revit-parameter-storage-types` 的区别: 本 skill 是"数值的单位语义"，storage-types 是"参数用什么数据类型存"——读取时先看 StorageType 再谈单位转换。
- 与批次7参数专题的区别: 本 skill 是单位换算这一横切规则，参数专题是参数读写全流程。
- 与 `revit-indexed-property-get-set` 的区别: 那是调用语法（get_Parameter），本 skill 是取到值之后怎么理解单位。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断数值的流向**
   - 完成标准: 确定这个数值是"从 API 读出（内部单位）"还是"用户输入要写入（显示单位）"。读 → 步骤 2；写 → 步骤 3。

2. **读取侧换算**
   - 完成标准: 展示用 `AsValueString()`（已格式化）或 `ConvertFromInternalUnits(value, DisplayUnitType)`；需要裸内部值（参与几何计算）则保留 `AsDouble()` 原样。确认没有把英尺裸值直接拼给用户。

3. **写入侧换算**
   - 完成标准: 用户输入先 `ConvertToInternalUnits(value, DisplayUnitType)` 再 Set；或 `SetValueString` 让 API 按项目单位解析。创建几何的尺寸参数确认填内部单位。验证写入后读回/界面显示一致。
   - 判停条件: 若参数是整数/无单位（如构件数量、YesNo）→ 停，不涉及单位换算，指路 storage-types。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 整数/枚举/ElementId 参数——没有单位概念。
- 字符串参数（文本）——无单位。

### 作者在书中警告的失败模式

- 直接把 AsDouble 的英尺值显示给用户 → 数字"看起来完全不对"。
- 把用户输入的毫米值直接 Set → 被当成英尺写入，模型尺寸错 25.4 倍量级。
- 手写长度换算公式（单位系混合）→ 导出单位非标准，推导必错。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：`UnitUtils.ConvertFromInternalUnits` 在 2014 用 `DisplayUnitType`，新版改为 `ForgeTypeId`（2021+）且 API 位置有调整，旧代码要迁移。
- 书未详述 `UnitFormatUtils`（新版本的完整格式化 API）与单位组（UnitGroup）管理，实操需补。

### 容易混淆的邻近方法论

- `AsValueString`（项目单位格式化字符串）vs `AsDouble`（内部单位裸值）——同为取值，语义完全不同。
- 内部单位（存储）vs 显示单位（UI 格式化）vs 项目单位（用户配置）——三个概念，转换只发生在 API 与 UI 的边界。

---

## 相关 skills

- revit-parameter-storage-types：composes-with——读取前先看 StorageType 定 Get/Set 方法，再看数值单位语义做换算，先类型后单位配套使用。
- revit-indexed-property-get-set：composes-with——get_Parameter 调用语法负责取到参数对象，本 skill 负责取到值之后怎么理解单位。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
