---
name: revit-settings-api-mapping
description: |
  当需要通过 API 程序化读取或修改项目级配置（材料、单位、阶段、类别、项目信息等）时调用。
  不适用于：图元级参数读写、视图级设置。
  关键 trigger 信号："项目设置 project settings"、"材料定义 materials"、"项目单位 units"、"参数绑定 parameter bindings"。
  核心映射：Document.Settings.Materials/Categories + Phases/ProjectInformation/ParameterBindings/GetUnits()。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第1章 1.4.4 (约p063-064)
tags: [settings-mapping, project-configuration, materials, units, revit-api]
related_skills: []
---

# Settings 对象映射到项目信息、参数绑定、材料、阶段、单位等

## R — 原文 (Reading)

> 表 1-11：设置 ➤ 项目信息 → Document.ProjectInformation；设置 ➤ 材料 → Document.Settings.Materials；阶段信息 → Document.Phases；项目单位 → Document.GetUnits()。
>
> — 宦国胜, 第1章 1.4.4 (约p063-064)

---

## I — 方法论骨架 (Interpretation)

Revit UI 中"管理/设置"对话框下的大部分项目级配置，都可以通过 Document 对象的属性程序化访问。这构成一张固定的映射表：

- **项目信息** → `Document.ProjectInformation`（项目名称、编号、地址等）
- **参数绑定** → `Document.ParameterBindings`（仅对共享参数的类别绑定）
- **项目位置** → `Document.ProjectLocations` / `Document.ActiveProjectLocation`
- **材料** → `Document.Settings.Materials`（全部材料定义）
- **对象样式/类别** → `Document.Settings.Categories`（类别与子类别）
- **阶段** → `Document.Phases`（项目阶段列表）
- **单位** → `Document.GetUnits()`（项目单位设置）

关键认知：这些不是分散的 API 调用，而是 UI"设置"对话框的 API 镜像。插件可以读取或修改项目级配置而无需用户手动打开对话框。其中材料热属性（ThermalProperties）和阶段属性是更深层访问点。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 通过 API 修改项目所有材料定义

- **问题**: 需要批量读取或修改项目的材料定义，不想让用户手动打开"材料"对话框
- **方法论的使用**: 查 Settings 映射表 → Document.Settings.Materials 即可获取全部材料定义的句柄
- **结论**: 材料定义可通过 API 程序化访问和修改
- **结果: 插件批量处理材料属性而无需用户干预

### 案例 2: 获取项目单位设置

- **问题**: 需要知道项目当前使用的长度、面积等单位
- **方法论的使用**: 查映射表 → Document.GetUnits() 返回 Units 对象
- **结论**: 单位信息可通过 API 读取，用于格式化输出或单位转换
- **结果**: 插件正确格式化数值输出，匹配项目单位

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件需要批量读取或修改项目级配置（材料、单位、阶段等）而不弹对话框
2. 需要知道某个"设置"面板的 UI 操作对应哪个 API 属性
3. 构建项目级报告工具，需要汇总材料、类别、阶段信息

### 语言信号 (用户的话里出现这些就应激活)

- "项目设置 / project settings / 项目配置"
- "材料定义 / materials / 材料属性"
- "项目单位 / units / 单位设置"
- "参数绑定 / parameter bindings / 对象样式 / categories"

### 与相邻 skill 的区分

- 与 `revit-binding-type-vs-instance` 的区别：该 skill 关注绑定的类型/实例二分；本 skill 是 Settings 对象到 API 属性的整体映射表入口。
- 与 `revit-shared-parameter-file-init` 的区别：该 skill 关注共享参数文件路径初始化；本 skill 关注 Settings 各集合的映射关系。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **查 Settings 映射表，定位目标属性**
   - 材料 → `Document.Settings.Materials`
   - 类别 → `Document.Settings.Categories`
   - 阶段 → `Document.Phases`
   - 单位 → `Document.GetUnits()`
   - 项目信息 → `Document.ProjectInformation`
   - 参数绑定 → `Document.ParameterBindings`
   - 项目位置 → `Document.ProjectLocations`
   - 完成标准: 确定目标设置对应的 API 属性路径

2. **通过该属性读取或修改数据**
   - 修改操作须在 Transaction 内执行
   - 完成标准: 成功读取或写入目标配置
   - 判停条件: 若属性返回 null，检查文档是否已加载对应设置

3. **验证修改是否生效**
   - 完成标准: 重新读取确认值已变更

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 图元级参数读写——用 Element.get_Parameter / LookupParameter，不走 Settings
- 视图级设置（如视图范围、可见性图形替换）——走 View 对象属性，不在 Settings 映射内

### 作者在书中警告的失败模式

- ParameterBindings 只对共享参数有效——项目参数和内置参数不走此路径

### 作者的盲点 / 时代局限

- 书基于 Revit 2014，后续版本可能新增 Settings 子属性（如分析模型设置），映射表需补充

### 容易混淆的邻近方法论

- Document.Settings（项目级设置）vs Element.Parameters（图元级参数）——前者是全局配置，后者是图元实例数据

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
