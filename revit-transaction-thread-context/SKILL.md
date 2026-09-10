---
name: revit-transaction-thread-context
description: |
  在后台线程或非模态对话框写模型时使用。规则：事务只能从受支持工作流启动（外部命令/事件/更新器/回调）；线程外或非模态对话框外启动抛异常。何时调用：后台任务写回结果；非模态 UI 按钮改图元。何时不调用：模态对话框内；主线程命令中。Trigger：'background thread / cross-thread / non-modal dialog / InvalidOperationException / Task.Run 写模型'。解法：IExternalEventHandler + ExternalEvent.Raise()。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2 开头（约p325）
tags: [threading, transaction, external-event, modal-dialog, thread-affinity]
related_skills:
  - slug: revit-transaction-off-thread
    relation: composes-with
---

# 事务不能从线程外或非模态对话框外启动

## R — 原文 (Reading)

> 注：若从线程以外或非模态对话框以外启动事务，则将引发异常。事务只能从受支持的 API 工作流中启动，如部分外部命令、事件、更新器或者回调。
>
> — 宦国胜, 第5章 5.2 开头（约p325）

---

## I — 方法论骨架 (Interpretation)

Revit 的模型对象有**线程亲和性（thread-affinity）**：`Document`、图元、事务只能在 Revit 主线程（UI 线程）的受支持上下文中访问。这条规则把你能写模型的入口限定为几类：**外部命令的 Execute、事件回调、更新器（IUpdater）Execute、以及作为回调被调用的代码**。

两个常见的"看起来能写但实际不能写"的地方：

1. **非主线程**：后台线程（Task.Run、Thread、async 续体）里访问 Document 或启动事务 → 抛异常。计算可以放后台，**写模型必须回主线程**。
2. **非模态对话框之外**：WPF/WinForms 的**非模态**窗口（Show()，不阻塞主线程）属于另一个消息上下文，它的按钮点击回调里也不能直接开事务。**例外是模态对话框**（ShowDialog()，阻塞主线程直到关闭）——模态期间 UI 线程仍受控，可以开事务。

当从非模态 UI 或后台线程发起"改模型"时，标准桥接是 **External Events 框架**：把修改逻辑写进 `IExternalEventHandler.Execute()`，用 `ExternalEvent.Create(handler)` 得到事件，再从任意线程调 `event.Raise()`——Revit 会在主线程空闲时执行 handler，事务在 handler 里正常开。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: WPF 非模态面板按钮创建墙
- **问题**: V2 预测场景——非模态面板上按钮点击后要创建一面墙，直接 new Transaction().Start() 会怎样？
- **方法论的使用**: 判定上下文——非模态对话框线程不是 Revit 主线程上下文，违反 thread-affinity。
- **结论**: 直接抛 InvalidOperationException。
- **结果**: 正确做法是把"创建墙"封装为 IExternalEventHandler.Execute()，经 ExternalEvent.Raise() 投递到主线程空闲时执行。

### 案例 2: 非模态对话框的 External Events 桥接
- **问题**: 第5章 5.4 专门讲非模态对话框与主线程的通信。
- **方法论的使用**: 用 External Events 框架在非模态 UI 与 Revit 主线程之间架桥。
- **结论**: 所有跨线程/跨非模态上下文的模型操作统一走 handler + Raise()。
- **结果**: 插件 UI 可以在非模态面板上自由按钮，模型修改安全落地。

### 案例 3: 事件回调与更新器中的事务限制
- **问题**: 第5章 5.2.2 事件回调中事务行为受限；第5章 5.6.2 IUpdater Execute 内不能开新事务。
- **方法论的使用**: 意识到"受支持的 API 工作流"内部还有各自的事务规则。
- **结论**: 即使在主线程，事件/更新器上下文也不是随便开事务。
- **结果**: 在这些回调里要么不开事务，要么按各自规则排队/延迟。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件用后台线程跑耗时计算（点云匹配、结构分析），算完要把结果写回模型。
2. WPF 非模态面板/侧边栏上有按钮，点了要改图元，直接写抛异常。
3. 用户报"按钮点了没反应 / 有时报 InvalidOperationException"。
4. async/await 之后代码里还拿着 Document 想写模型。

### 语言信号 (用户的话里出现这些就应激活)

- "后台线程里能开事务吗？" / "start a transaction from a background thread"
- "跨线程访问 Revit 模型" / "cross-thread access to the revit model"
- "非模态对话框里写模型" / "write model from a modeless dialog"
- "Task.Run 里改图元" / "modify elements inside Task.Run"
- "ExternalEvent 怎么用？" / "how to use ExternalEvent and Raise"

### 与相邻 skill 的区分

- 与 `revit-transaction-off-thread`（revit-transaction-off-thread）：那是本规则的反例镜像——从违反场景出发讲"会抛什么异常、后果是什么"；本 skill 从正面讲规则与桥接方案。两者互为表里。
- 与 `revit-transaction-hierarchy`（revit-transaction-hierarchy）：那个讲事务的层级结构；本 skill 讲启动事务的**上下文**门槛，先过上下文关再谈层级。
- 与 `revit-failures-processing-event`（revit-failures-processing-event）：事件机制不同——FailuresProcessing 是故障引擎的回调；本 skill 的"事件"是 External Event 桥接机制。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断当前上下文**：问"这段写模型的代码跑在哪个线程/哪种 UI 上下文？"
   - 在外部命令 Execute / 模态对话框内 / 受支持事件回调 → 可直接开事务。
   - 在后台线程或非模态对话框回调 → 进入步骤 2。
   - 完成标准: 明确归类为"可直接写"或"必须桥接"。
   - 判停条件: 若可直接写，跳过桥接，直接按事务规范执行。

2. **搭 External Event 桥**：
   - 实现 `IExternalEventHandler`：`Execute(UIApplication app)` 里写完整的事务逻辑（包括 try-catch 与回滚）。
   - 用 `ExternalEvent.Create(handler)` 创建事件对象并保存引用。
   - 在后台/非模态代码处调用 `event.Raise()`（Raise 可从任意线程调用，不阻塞）。
   - 完成标准: handler 已实现、事件已创建、Raise() 在正确位置被调用。

3. **验证执行结果**：
   - 在 handler 内做结果回传（通过 handler 上的字段/回调，或 ExternalEvent.Raise 后续轮询）。
   - 确认模型修改实际发生在主线程空闲期，且事务状态正确。
   - 完成标准: 修改生效、无跨线程异常；若 Raise 后 handler 未执行，检查事件是否被外部关闭或未正确处理。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 已处于 Revit 主线程的模态对话框或命令上下文中——直接开事务即可，过度桥接反而复杂。
- 纯计算不碰 Document——后台线程随便跑，无需桥接。
- 只想读模型数据（不写）——部分只读 API 在线程亲和上有放宽，但 Document 对象本身仍不宜跨线程持有；保守起见仍回主线程。

### 作者在书中警告的失败模式

- **线程外启动事务直接抛异常**（revit-transaction-off-thread 完整反例）。
- **非模态对话框之外启动事务抛异常**——注意"模态可以、非模态不行"的反差。
- **IUpdater Execute 内不能开新事务**（第5章 5.6.2）——更新器上下文再特殊。
- **事件回调中事务行为受限**（第5章 5.2.2）——别在事件里顺手开事务。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：External Event 的 `Raise()` 在后续版本支持了带参数回调（Raise 失败原因），错误诊断更友好，可用新版增强。
- 未深入讨论 async/await 下的上下文捕获问题——同步上下文（SynchronizationContext）在 Revit 里的行为因宿主不同有差异，不要假设 await 后一定回到主线程。
- 未覆盖多文档（Document 切换）场景下的线程上下文。

### 容易混淆的邻近方法论

- 模态 vs 非模态对话框：**模态（ShowDialog）能开事务，非模态（Show）不能**——这是最容易踩错的反差规则。
- External Event vs 普通事件（如 DocumentChanged）：前者是把代码"投递"到主线程执行（可开事务）；后者是只读通知（不可开事务）。

---

## 相关 skills

- revit-transaction-off-thread：composes-with——本 skill 从正面讲线程/上下文规则与 External Event 桥接，off-thread 从反面讲违反场景的异常与后果，互为表里构成完整认知。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
