---
name: revit-command-state-persistence
description: |
  当问'命令执行完上次的状态怎么保存'、或命令类字段下次调用时值丢了时调用。命令对象 Execute 返回即销毁，字段无法自然持久化。正解：跟文档走→共享参数/Extensible Storage；会话
  级缓存→IExternalApplication 静态字段；重数据→外部文件/数据库。不适用：单次命令内局部状态。trigger：状态丢失、保存配置、shared parameters、state p
  ersistence。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p032
tags: [counter-example, state-persistence, shared-parameters, extensible-storage]
related_skills:
  - slug: revit-command-app-loading-timing
    relation: composes-with
  - slug: revit-data-storage-paths
    relation: composes-with
---

# 外部命令执行完毕即销毁，状态无法自然持久化

## R — 原文 (Reading)

> 当此方法返回 Revit，命令对象即被销毁。因此，命令执行之间的对象数据无法存留。不过，有其他方法可以保存命令执行之间的数据。例如，可以使用 Revit 共享参数机制来存储 Revit 项目数据。
>
> — 宦国胜, 第1章 1.3.2（约 p032）

---

## I — 方法论骨架 (Interpretation)

这个反直觉事实是新手最容易栽的坑：**命令对象的生命周期只有一次 Execute 调用那么长**。

- Revit 在点击时 new 你的命令类 → 调 `Execute` → 返回后**销毁实例**。类里定义的字段（List、Dictionary、字符串）全部随之消失。
- 所以"把上次的选择存进字段，下次命令再读"是无效的——每次 Execute 拿到的都是全新对象。

要跨命令保存状态，Revit 给出了分层方案：

1. **随模型持久**（换电脑、关 Revit 都在）：轻量配置 → **共享参数**（Shared Parameters）；结构化数据 → **Extensible Storage**（挂在图元/文档上的自定义 Schema）。
2. **会话级驻留**（Revit 开着就在，关了没）：`IExternalApplication` 的静态字段/缓存——Application 全程驻留，是"运行时内存状态"的家。
3. **进程外**：外部文件、数据库——重数据或需多机共享时。

选择判据：**状态跟谁走**。跟文档走 → 共享参数/Extensible Storage；跟本次 Revit 会话走 → Application 静态字段；跟机器/团队走 → 外部存储。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 命令对象销毁（1.3.2）
- **问题**: 想在多次命令调用间记住用户上次选的图层，字段存法可行吗？
- **方法论的使用**: 认识到命令对象返回即销毁，字段不可靠；转向共享参数机制。
- **结论**: 状态必须存在"活得比命令长"的地方。
- **结果**: 书中明确以共享参数作为跨命令存 Revit 项目数据的官方路径。

### 案例 2: Application 驻留作为替代（1.3.3）
- **问题**: 运行时缓存放哪？
- **方法论的使用**: 用 IExternalApplication 提供驻留生命周期，静态字段承载缓存。
- **结论**: 会话级状态归 Application 管。
- **结果**: 事件/动态更新等需跨命令状态的场景有了可靠载体。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 命令第一次运行设置了某值，第二次运行发现字段是空的、一切重来。
2. 想让"用户上次选的选项/颜色/图层"在下次打开命令时恢复。
3. 分不清共享参数、Extensible Storage、外部文件分别该存什么。
4. 写了"静态字段"但 Revit 重载/重开后丢了，想知道为什么。

### 语言信号 (用户的话里出现这些就应激活)

- "命令之间状态丢失 / 字段被清空"
- "保存用户配置 / 记住上次选择"
- "共享参数还是 Extensible Storage"
- "跨命令怎么存数据"
- "state lost between commands / persist settings / shared parameters / extensible storage / static cache"

### 与相邻 skill 的区分

- 与 `revit-command-app-loading-timing` 的区别: 本 skill 是"命令对象被销毁→状态没地方住"；loading-timing 讲 Application 驻留因此是缓存的家——两者互补。
- 与 `revit-data-storage-paths` 的区别: 本 skill 是"选存储位置的决策"，data-storage-paths 是共享参数/Extensible Storage 的具体读写 API。
- 与 `revit-db-application-scenarios` 的区别: DBApplication 提供的驻留同样是缓存宿主，但那是无 UI 后台场景。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位状态的真实寿命需求**
   - 完成标准: 回答"这个状态要活多久、跟谁走"：跟文档走 / 跟本次会话走 / 跟机器团队走。若用户说"要永久记住"→ 跟文档或外部走；说"只要 Revit 开着就行"→ 会话级。

2. **按寿命选存储方案**
   - 完成标准: 文档级轻量 → 共享参数；文档级结构化 → Extensible Storage；会话级缓存 → IExternalApplication 静态字段；重数据/多机 → 外部文件或数据库。对用户解释各方案写入/读取的大致代码入口，并说明静态字段在 Revit 重启后失效。
   - 判停条件: 若用户只是单次命令内的临时变量，停——不需要持久化，别过度设计。

3. **验证跨命令读取**
   - 完成标准: 第一次运行写入、第二次运行读到同一值；重启 Revit 后按预期保留（文档级）或清空（会话级）。给用户列出"命令字段×寿命"对照结论。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 单次命令内的局部变量——不需要任何持久化机制。
- 正在写共享参数/Extensible Storage 的具体代码——那是数据存储专题。

### 作者在书中警告的失败模式

- 依赖命令类字段存状态 → 每次执行都"失忆"，行为随缘。
- 静态字段缓存未做失效处理 → Revit 会话内数据过期（文档已切换）。
- 把大数据塞共享参数 → 模型文件膨胀、跨图元传播性能差。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：Extensible Storage 在新版仍是首选但 Schema 结构约束更严；书对共享参数着墨多、对 ES 较简略。
- 书未讨论 Revit 崩溃/版本升级时 Extensible Storage 的兼容问题。

### 容易混淆的邻近方法论

- 共享参数（项目级参数、需定义文件）vs Extensible Storage（挂在元素上的私有 Schema）——前者用户可见可排入明细表，后者程序私有。
- "静态字段"（AppDomain 内存，会话级）vs "配置文件"（磁盘，跨会话）——两者寿命不同，别混用。

---

## 相关 skills

- revit-command-app-loading-timing：composes-with——本 skill 讲"命令对象销毁后状态没地方住"，loading-timing 讲 Application 驻留因而是会话缓存的宿主，两者互补构成完整方案。
- revit-data-storage-paths：composes-with——本 skill 是"按寿命选存储位置"的决策，data-storage-paths 是共享参数/Extensible Storage 的具体读写路径。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
