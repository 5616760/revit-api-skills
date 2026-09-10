---
name: revit-id-vs-uid-decision
description: |
  附录术语版双 ID 决策：ElementId（整数，项目内唯一）vs UniqueId（全局唯一 GUID）。外部数据库/BIM 导出、工作共享场景必用 UID——同步后 ElementId 可能变化；复制时 UID 必变而 ElementId 不一定变；Updater 管理建议用 UID。信号："导出图元 ID / export element ids"、"工作共享 ID 变化 / element id changes after sync"、"UID 是什么 / what is uid"。与 revit-elementid-vs-uniqueid 同源，待合并。
  Trigger: element id / uid / guid / worksharing sync。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录A 术语 / 附录B FAQ（约p425-426）
tags: [revit-api, elementid, uniqueid, glossary, worksharing]
related_skills:
  - slug: revit-elementid-vs-uniqueid
    relation: contrasts-with
---

# 元素 ID vs 元素 UID 决策（附录术语版）

## R — 原文 (Reading)

> 图元 ID：每个图元都对应一个 ID，用一个整数值加以标识。在 AutoCAD Revit 项目中，它提供了一种唯一地识别图元的方式。
> 图元 UID：每个图元都对应有一个 UID，它是一串全局唯一的标识符。这意味着即使跨不同 AutoCAD Revit 项目，图元 UID 也是唯一的。
>
> — 宦国胜, 附录A 术语 / 附录B FAQ（约p425-426）

---

## I — 方法论骨架 (Interpretation)

这是正文书"双 ID 体系"在附录术语层的浓缩版，补充了三个工程视角：

1. **基础定义**：图元 ID（ElementId）是整数，唯一性以"本项目"为界；图元 UID（UniqueId）是 GUID 字符串，全局唯一。外部系统引用一律用 UID。
2. **动态稳定性差异**（附录证据的增量）：
   - 工作共享（Worksharing）同步时，ElementId **可能变化**——中心文件同步是 ID 漂移的真实来源，依赖 ElementId 做跨同步跟踪会断链。
   - 复制图元时，UID **必然变化**（保证全局唯一），ElementId 不一定变——"复制后 ID 对不上/撞上"要用这套规则解释。
3. **框架级建议**：更新器（Updater/动态更新框架）管理中建议用 UniqueId 而非 ElementId——凡是要在"时间维度上持续引用"的场景都倾向 UID。

一句话决策：读一次就用 → ID；存下来以后用/跨人跨文件用 → UID。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: BIM 数据管道导出
- **问题**: 把 Revit 图元 ID 导出到外部数据库做集成。
- **方法论的使用**: 按附录 FAQ 的唯一性边界判断——ElementId 跨项目不唯一 → 导出字段用 UID。
- **结论**: 管道主键 = UniqueId。
- **结果**: 外部记录可跨项目、跨时间可靠回指。

### 案例 2: 工作共享同步的 ID 漂移
- **问题**: 协作模型每次与中心文件同步后，外部记录的图元 ID 部分失配。
- **方法论的使用**: 用"工作共享同步时 ElementId 可能变化"解释失配，改为 UID 键。
- **结论**: 多人协作场景禁用 ElementId 做持久键。
- **结果**: 同步后引用不断链。

### 案例 3: 更新器管理的建议
- **问题**: 动态更新框架中标识被更新的图元。
- **方法论的使用**: 采纳书中建议——更新器管理用 UniqueId。
- **结论**: 框架级持续引用场景也属 UID 领地。
- **结果**: 更新跟踪在会话间保持稳定。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 快速查术语："图元 ID 和 UID 有什么区别"这类概念问答。
2. 工作共享环境下外部记录与模型对不上号（同步后 ID 漂移）。
3. 复制操作后观察到的 ID 行为异常（UID 变了/ID 没变）需要解释。
4. 为更新器、外挂数据库、对量工具选长期主键。

### 语言信号 (用户的话里出现这些就应激活)

- "图元 ID / 图元 UID 是什么 / what is element uid"
- "工作共享同步后 ID 变了 / element id changed after sync"
- "导出 ID 到数据库 / export ids to database"
- "更新器用哪个 ID / updater uniqueid"

### 与相邻 skill 的区分

- 与 `revit-elementid-vs-uniqueid` 的区别: 正文版给完整决策框架（含 IntegerValue 比较、IFC 跟踪）；本 skill 是附录术语浓缩 + 工作共享/复制稳定性/更新器三个增量证据。两者同源，阶段 3 应合并或以正文版为主、本版补充证据。
- 与 `revit-element-retrieval-four-entries` 的区别: 本 skill 只管标识选型；按 ID 检索的入口选择属四法决策。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断引用的时间跨度**：本次运行 / 跨会话 / 跨项目 / 跨协作同步？
   - 完成标准: 跨度结论明确。
   - 判停条件: 若仅本次运行 → 直接用 ElementId，结束。

2. **跨时间引用一律 UID**：外部数据库、更新器、协作跟踪的主键统一 UniqueId；同时校验该场景是否存在工作共享同步（ID 漂移风险最高的场景）。
   - 完成标准: 持久化字段为 UID；工作共享项目中被确认为不依赖 ElementId。

3. **解释既有异常（如有）**：用"复制时 UID 必变、ElementId 不一定变；同步时 ElementId 可能变"两条规则诊断 ID 类失配报告。
   - 完成标准: 异常被归因到具体规则，并给出迁移方案（ElementId → UID + 映射表）。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要完整决策框架与代码路径（GetElement/IdSet）——转正文版 `revit-elementid-vs-uniqueid`。
- 讨论检索入口选择——转四法决策 skill。

### 作者在书中警告的失败模式

- 跨项目复用 ElementId 当唯一键——撞号不可避免。
- 工作共享模型用 ElementId 做外部跟踪——同步即断链。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：附录描述的规则稳定；新版对链接文档中 UID 的表达（含链接实例标识的组合形式）有更细的语义，跨链接跟踪需查新版文档。

### 容易混淆的邻近方法论

- "复制后 UID 变"常被误读为"UID 不稳定所以不能用"——恰恰相反：UID 变是因为新对象是新身份，这正是它作为身份标识的正确行为；不稳定的是跨同步的 ElementId。

---

## 相关 skills

- revit-elementid-vs-uniqueid：contrasts-with——同源双版本：本 skill 是附录术语速查版（补充工作共享/复制稳定性、Updater 建议），revit-elementid-vs-uniqueid 是正文完整版（含 IntegerValue 比较、IFC 跟踪），两者互为对照。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段4测试 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
