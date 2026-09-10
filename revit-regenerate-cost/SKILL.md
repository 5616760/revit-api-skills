---
name: revit-regenerate-cost
description: |
  批量修改性能优化时使用。规则：Regenerate() 高成本；提交事务时已自动重生成一次，所以批量修改应合并到大事务、只在必须读最新几何时手动调。何时调用：批量改大量参数、插件慢、Regenerate 调用过多。何时不调用：需中间读新几何。Trigger：'插件太慢/批量修改卡/Regenerate 太多'（regenerate is expensive, batch transactions slow）。1000 小事务=1000 次重生成。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 附录B FAQ（约p426）
tags: [performance, regenerate, batch, transaction, optimization]
related_skills:
  - slug: revit-transaction-mode-selection
    relation: depends-on
  - slug: revit-transaction-hierarchy
    relation: composes-with
---

# 不要在事务提交时频繁调用 Regenerate

## R — 原文 (Reading)

> 确保只在必须时才调用 Document.Regenerate()。尽管需要使用这种方法来确保 Revit 文件中的图元反映所有更改，但这会减慢应用程序的运行速度。还得记住，在提交事务时，会有一个重生成文件的自动调用。
>
> — 宦国胜, 附录B FAQ（约p426）

---

## I — 方法论骨架 (Interpretation)

`Document.Regenerate()` 很贵：它要沿参数化依赖图传播修改、重建几何、刷新分析模型。一次调用就是一次全模型级运算。Revit 里有两件容易忽视的事：

1. **提交事务时系统会自动重生成一次**。也就是说，你事务一 Commit，模型已经"焕然一新"了，不需要也不应该再手动补一次 Regenerate。
2. **手动 Regenerate 应该只在"必须"时调用**——这个"必须"通常只有一个含义：你需要在事务提交**之前**，立刻拿到依赖最新几何的计算结果（比如改完墙厚马上算体积）。

由此推出的性能原则：**批量修改尽量合并进一个大事务，而不是拆成很多小事务**。每个事务提交都会触发一次全模型重生成——改 1000 个墙、开 1000 个小事务，就是 1000 次重生成；合并成 1 个大事务只重生成 1 次，速度差数倍。这是 Revit 特有的"重生成是重量级操作"与普通数据库"每笔提交便宜"的直觉之间的关键差异。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量改墙参数的两种方案
- **问题**: V2 预测场景——批量修改 1000 个墙的参数，方案① 1000 个小事务各改一个，方案② 1 个大事务改全部，哪个快？
- **方法论的使用**: 应用"提交即自动重生成"与"Regenerate 高成本"两条规则。
- **结论**: 方案②快数倍——①触发 1000 次重生成，②只触发 1 次。
- **结果**: 应把多个修改合并到一个大事务；仅在必须手动修改后立即查询依赖几何时才调 Regenerate。

### 案例 2: 事务模式对重生成的影响
- **问题**: 外部命令的 TransactionMode 选择（第1章 1.3.7 / revit-transaction-mode-selection）——Automatic 模式下框架自动创建事务并提交。
- **方法论的使用**: Automatic 模式每个命令自动提交 = 每次命令结束都自动重生成。
- **结论**: 批量操作应选 Manual 模式，自己控一个大事务，避免被框架切成多次提交。
- **结果**: 大批量脚本性能显著提升。

### 案例 3: 与几何时序约束的平衡
- **问题**: 某些场景确实需要立即读新几何（revit-regenerate-geometry-timing）。
- **方法论的使用**: 权衡"Regenerate 高成本"与"几何无效"两个约束。
- **结论**: 仅在需要中间结果时手动 Regenerate，且尽量集中到少数关键点。
- **结果**: 在正确性与性能之间取得平衡。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 批量脚本（几百上千个图元）运行极慢，用户问"能不能优化"。
2. 代码里每个图元一改就调一次 Regenerate，需要评估是否多余。
3. 在"拆成小事务"与"合并大事务"之间做选择。
4. 需要解释为什么"提交事务时不用再手动 Regenerate"。

### 语言信号 (用户的话里出现这些就应激活)

- "批量修改太慢，怎么优化？" / "batch modification is slow, optimize it"
- "Regenerate 是不是不用调这么多次？" / "too many Regenerate calls"
- "1000 个小事务还是 1 个大事务？" / "one big transaction or many small ones"
- "提交事务会自动重生成吗？" / "does commit auto-regenerate"
- "性能优化，减少 Regenerate" / "reduce Regenerate calls for performance"

### 与相邻 skill 的区分

- 与 `revit-transaction-mode-selection` 的区别: 本 skill 的"合并大事务"策略依赖 Manual 模式自管事务，transaction-mode-selection 讲如何选择 Automatic/Manual/ReadOnly 档位。
- 与 `revit-transaction-hierarchy` 的区别: 合并事务的动机通常来自性能（本 skill），而层级结构是手段——两者经常一起用。
- 与 `revit-regenerate-failure-rollback`（revit-regenerate-failure-rollback）：那是失败恢复；本 skill 是性能原则，关注"调用次数"而非"失败处理"。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **审计 Regenerate 调用点**：统计代码里 `Document.Regenerate()` 的出现次数与所在循环。
   - 完成标准: 列出每个调用点、所在循环的迭代次数、是否真的需要立即读新几何。

2. **按用途分类并精简**：
   - 事务提交后/末尾的 Regenerate → **删除**（提交已自动重生成）。
   - 循环内每个迭代都调且不读中间几何 → **上移出循环**或删除。
   - 确实要在提交前读新几何 → 保留，但确认是"最少次数"。
   - 完成标准: Regenerate 只在"提交前必须读新几何"的关键点保留。

3. **合并批量事务**：
   - 把多个小事务合并成一个大事务（必要时用 TransactionGroup 管理）。
   - 对照 TransactionMode：批量命令用 Manual 模式自行控制。
   - 完成后实测对比运行耗时。
   - 完成标准: Regenerate 次数降到必要最小；批量脚本耗时明显下降；几何正确性不受影响。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要在每次修改后立即读新几何的算法（如迭代式布局）——此时 Regenerate 是必要成本，别为性能牺牲正确性。
- 单个小修改——性能问题不成立，保持简单直接。

### 作者在书中警告的失败模式

- **频繁调用 Regenerate 拖慢应用**——这是本 skill 的核心警告，尤其在循环内。
- **忘记提交时自动重生成**——于是白白多调一次 Regenerate，白付一次高成本。
- **大事务的风险**：合并事务虽快，但事务越大，失败时回滚代价越大、对工作共享锁的占用越久——性能与风险的权衡。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版的重生成引擎（并发再生、后台再生）有优化，但"提交自动重生成"与"Regenerate 高成本"的基本盘不变。
- 未讨论"局部 Regenerate"的可能性——旧版 API 没有按图元细粒度重生成，都是文档级；新版本的部分 API 变化需查 SDK。
- 未量化"多少次调用算频繁"——阈值依赖项目规模，应实测而非猜。

### 容易混淆的邻近方法论

- "提交自动重生成" vs "手动 Regenerate"：前者免费（已含在提交里），后者要钱（单独触发）——不要把两者叠加。
- "合并事务提速" vs "事务越小越安全"：性能取合并，风险取拆分——按失败代价与锁竞争决策，不是单纯二选一。

---

## 相关 skills

- revit-transaction-mode-selection：depends-on——本 skill 的"合并大事务"方案依赖 Manual 模式自管事务，前提是先理解 TransactionMode 三档位的选型。
- revit-transaction-hierarchy：composes-with——本 skill 讲合并事务的性能动机，transaction-hierarchy 讲 Transaction/SubTransaction/TransactionGroup 的组合手段，动机与手段配套使用。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
