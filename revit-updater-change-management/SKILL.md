---
name: revit-updater-change-management
description: |
  编写会持久化图元引用或有多个更新器并存的 IUpdater 逻辑时调用。原则：跨图元/跨会话持久化
  引用必须用 Element.UniqueId（GUID）而非 ElementId（同步时会漂移）；更新器逻辑必须幂等，防止
  A 改 B、B 触发 A 的无限循环；同一图元可能被不同更新器在同一事务内修改，逻辑要能共存。
  Trigger："UniqueId vs ElementId"、"更新器循环触发"、"中心同步后引用失效"、"幂等"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.6.2（约p360-361）
tags: [dmu, uniqueid, idempotency, worksharing, revit-api]
related_skills:
  - slug: revit-iupdater-execute-transaction-rules
    relation: depends-on
  - slug: revit-updater-failure-modes
    relation: composes-with
  - slug: revit-elementid-vs-uniqueid
    relation: depends-on
---

# 更新器管理更改原则（避免重复触发与冲突）

## R — 原文 (Reading)

> "更新器必须能够处理在使用过程中可能引发的复杂问题……也有可能同一图元被不同的更新器修改，甚至可能发生在同一事务内部……当文件与其中心文件进行同步时，图元的 ElementId 可能会受到影响……应使用 Element.UniqueId，以保证唯一性。"
>
> — 宦国胜，第5章 5.6.2（约p360-361）

---

## I — 方法论骨架 (Interpretation)

更新器一旦"记住图元"或"修改图元"，就进入多更新器、多用户、可同步的复杂世界，需要三条纪律。

- **标识纪律**：ElementId 是项目内整数编号，与中心文件同步（合并、冲突解决）时可能被重新编号；跨图元、跨会话持久化引用必须用 `Element.UniqueId`（GUID，全局唯一稳定）。
- **幂等纪律**：更新器修改过的图元可能再次满足触发条件被重新触发。Execute 的逻辑必须幂等——同样输入产出同样结果、目标状态已达成则直接返回，否则 A 改 B、B 触发 A 的循环。
- **共存纪律**：同一图元可能被不同更新器在同一事务内修改。写逻辑时假设"图元可能已被别人改过"，先读当前状态再决定动作，不基于陈旧缓存做断言。
- 附带事实：图元的复制/粘贴会拷贝其附着数据——依赖附着数据做判断时要考虑副本场景。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 用 ElementId 持久化"被管过的图元"

- **问题**: 更新器用 ElementId 记录自己管理过的图元，项目与中心文件同步后引用失效。
- **方法论的使用**: 书中指出同步时 ElementId 可能受影响，应使用 Element.UniqueId 保证唯一性。
- **结论**: 任何要"存下来下次还认得"的引用，一律 UniqueId。
- **结果**: 同步/合并后引用依然有效，更新器状态不漂移。

### 案例 2: 两个更新器互相触发

- **问题**: 更新器 A 改图元 X，X 的变化触发更新器 B，B 又改回 X，形成循环。
- **方法论的使用**: 幂等纪律——Execute 先判断目标状态是否已达成，已达成则不动作。
- **结论**: 幂等是循环的天然断路器（Revit 还有兜底禁用机制，见 `revit-updater-failure-modes`）。
- **结果**: 第二轮触发时状态已满足，链路终止。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 更新器需要在外部（配置文件/共享参数/extensible storage）记住图元引用。
2. 多个更新器共存或团队插件叠加，出现互相干扰。
3. 中心文件同步后更新器行为错乱、引用丢失。

### 语言信号 (用户的话里出现这些就应激活)

- "UniqueId / ElementId 用哪个"（UniqueId vs ElementId）
- "更新器互相触发 / 死循环"（updater infinite loop）
- "同步后 图元引用 失效"（reference broken after sync with central）
- "更新器 幂等"（idempotent updater）

### 与相邻 skill 的区分

- 与 `revit-iupdater-execute-transaction-rules` 的关系：本 skill 讲跨触发、跨更新器、跨同步的管理层面原则；该 skill 讲单次 Execute 的事务环境，本 skill 建立在其之上。
- 与 `revit-updater-failure-modes` 的关系：本 skill 是开发者侧的事前预防纪律（UniqueId/幂等）；该 skill 是 Revit 侧故障-恢复行为手册，二者互补。
- 与 `revit-elementid-vs-uniqueid` 的关系：跨更新器追踪必须用 UniqueId，复用该 skill 对 ElementId/UniqueId 场景的区分。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **审计所有持久化引用**
   - 搜索代码/存储中所有 ElementId 的持久化使用点，替换为 UniqueId。
   - 完成标准: 存储层（文件/数据库/extensible storage）只出现 UniqueId；运行时需要 ElementId 时由 UniqueId 查回。

2. **给 Execute 加幂等守卫**
   - 每个修改动作前判断"目标状态是否已是期望值"，是则跳过。
   - 完成标准: 对同一变更重复执行 Execute，第二次不产生实际修改。

3. **做共存假设检查**
   - 逻辑中所有"读到的图元状态"都按"可能被其他更新器改过"处理，先刷新再判断。
   - 完成标准: 无基于外部缓存的图元状态断言。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单用户、无工作共享、单更新器的小工具——纪律仍有益但不是阻塞项，按简单优先。
- 事务内的临时引用（同一 Execute 生命周期内）——ElementId 完全可用且更快。

### 作者在书中警告的失败模式

- 同步时 ElementId 漂移 → 持久化引用失效（本单元本体）。
- 同一图元被多个更新器修改、甚至同一事务内 → 非幂等逻辑产生不可预测结果。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014；新版 DMU 对循环检测与 ChangePriority 调度更精细，但幂等/UniqueId 纪律仍是根本。

### 容易混淆的邻近方法论

- 附录 A 的 ElementId vs UniqueId 对比是本原则的标识论基础。
- 复制/粘贴拷贝附着数据：Unique 也可能随复制产生新值（副本是新实体），设计引用时要考虑副本语义。

---

## 相关 skills

- **revit-iupdater-execute-transaction-rules**（IUpdater Execute 方法约束与事务规则 · depends-on）— 跨触发/同步的更改管理建立在单次 Execute 事务规则之上。
- **revit-updater-failure-modes**（更新器常见故障（无限循环、冲突编辑、中心文件） · composes-with）— 开发者侧预防纪律与该 skill 的 Revit 侧故障处置手册互补。
- **revit-elementid-vs-uniqueid**（ElementId 与 UniqueId 的使用场景与比较 · depends-on）— 跨更新器/跨同步的追踪依赖 UniqueId 语义，复用该 skill 的 Id 比较知识。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
