---
name: revit-transaction-off-thread
description: |
  反例：从后台线程/非模态对话框外启动事务，识别与修复。何时调用：Task.Run/async/Thread 里 new Transaction 抛 InvalidOperationException；非模态按钮点了没反应。何时不调用：已走 External Event 桥。Trigger：'start transaction from thread / cross-thread exception / 后台线程开事务报错'。修复=把写模型逻辑移入 IExternalEventHandler.Execute + ExternalEvent.Raise()；模型对象有线程亲和性。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第5章 5.2 开头（约p325）
tags: [threading, counter-example, transaction, external-event, exception]
related_skills: []
---

# 反例：从线程外或非模态对话框外启动事务

## R — 原文 (Reading)

> 注：若从线程以外或非模态对话框以外启动事务，则将引发异常。事务只能从受支持的 API 工作流中启动，如部分外部命令、事件、更新器或者回调。
>
> — 宦国胜, 第5章 5.2 开头（约p325）

---

## I — 方法论骨架 (Interpretation)

这是一个典型的"反例"单元：把"线程外启动事务"当作反面教材，讲它为什么失败、失败后怎么改。

失败根源：Revit 的 `Document` 和模型对象只在**主线程**上合法可访问（线程亲和）。你在 `Task.Run`、`Thread.Start`、async 续体、或非模态窗口的回调里 `new Transaction(doc).Start()`，等于让一个陌生线程去动主线程的对象——系统直接抛 `InvalidOperationException`（跨线程访问非法），有时表现为按钮点击后"什么都没发生"或偶发崩溃。

这个反例的价值在于它**强制了插件的架构形态**：耗时计算放后台（这是合理的），但"把结果写回模型"必须回到主线程。唯一的官方桥是 External Events 框架——把写模型逻辑封装进 `IExternalEventHandler.Execute()`，用 `ExternalEvent.Create(handler).Raise()` 投递到主线程空闲期执行。于是"后台任务 + 写模型"的插件，无论 UI 是 Ribbon 按钮还是非模态面板，最终都会长成同一个形状：**计算在后台，写入走 handler**。

注意这里的坑有多深：就算你让后台线程只调用"看起来像主线程代码"的方法，只要那个方法内部 `new Transaction`，照样抛异常——问题不在调用链的深浅，在于执行线程本身。检测信号就是任何跨线程持有的 `Document` 引用：别存字段、别传参数到后台。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 后台计算完成后写回模型
- **问题**: V2 预测场景——后台线程用 Task.Run 跑耗时计算，想在结果出来后用另一个线程开事务写回模型。
- **方法论的使用**: 识别线程亲和约束——任何跨线程访问都非法。
- **结论**: 直接抛 InvalidOperationException。
- **结果**: 唯一正确路径是 IExternalEventHandler.Execute + ExternalEvent.Raise()，把"写回"投递到 Revit 主线程空闲期执行。

### 案例 2: 非模态对话框按钮直写模型
- **问题**: WPF 非模态面板按钮点击回调里直接 new Transaction()，异常或无效。
- **方法论的使用**: 判定非模态对话框线程不是受支持的 API 工作流。
- **结论**: 必须桥接 External Event（第5章 5.4 框架）。
- **结果**: 按钮逻辑改为 handler.Execute 中的事务，UI 正常响应。

### 案例 3: IUpdater Execute 里开新事务
- **问题**: 第5章 5.6.2 更新器 Execute 内尝试开新事务。
- **方法论的使用**: 更新器虽在受支持工作流内，但**内部不能再开新事务**。
- **结论**: 事务启动上下文比"是否主线程"更严格——工作流内部还有各自的规则。
- **结果**: 更新器内的模型修改要么不进行，要么走允许的通道。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 插件代码在 Task.Run / async / Thread 里 new Transaction，用户报"抛异常/闪退"。
2. 非模态面板上按钮点了没反应或偶发报错，怀疑线程问题。
3. 用户问"为什么不能在后台线程改模型？我计算的线程不够用"。
4. 已有代码把 Document 存成静态字段/后台引用，需要审计。

### 语言信号 (用户的话里出现这些就应激活)

- "后台线程开事务报错" / "starting transaction from background thread fails"
- "跨线程访问异常 / InvalidOperationException" / "cross-thread access exception"
- "Task.Run 里写模型可以吗？" / "write the model inside Task.Run"
- "非模态窗口按钮点了没反应" / "modeless dialog button does nothing"
- "为什么线程外不能开事务" / "why can't I start a transaction off-thread"

### 与相邻 skill 的区分

- 与 `revit-transaction-thread-context`（revit-transaction-thread-context）：正反镜像。p06 讲规则和正确的桥接方案；本 skill 讲反例——你看到的就是"线程外开事务失败"这个具体 bug 的识别与修复。
- 与 `revit-transaction-hierarchy`（revit-transaction-hierarchy）：层级结构是"怎么组织事务"，线程上下文是"在哪里能启动事务"，先过了本 skill 的上下文关再谈结构。
- 与 `revit-failures-processing-event`（revit-failures-processing-event）：别把 External Event 和 FailuresProcessing 事件混了——前者桥接线程，后者处理故障。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位线程违规点**：搜索 `new Transaction`、`doc.Commit`、跨线程的 `Document` 引用，判断它们所在的执行上下文（线程/对话框）。
   - 完成标准: 列出所有"非主线程/非模态上下文里触碰模型"的代码点。
   - 判停条件: 若未发现违规点、异常另有原因（如事务未 Start 就 Commit），停下转向其他诊断。

2. **把写模型逻辑搬进 handler**：
   - 新建类实现 `IExternalEventHandler`，在 `Execute()` 里写完整事务逻辑（Start/Commit/RollBack + 异常处理）。
   - 用 `ExternalEvent.Create(handler)` 建事件，保存引用。
   - 原违规点改为调用 `event.Raise()`。
   - 完成标准: 原位置不再有任何 `new Transaction`；所有模型写入集中在 handler.Execute()。

3. **验证与回传**：
   - 运行验证：Raise 后模型修改生效、无跨线程异常。
   - 需要回传计算结果时，在 handler 上设字段/回调供 UI 读取。
   - 完成标准: 后台计算→Raise→主线程写模型 全链路跑通；无 Document 被跨线程持有。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 代码本来就合规（无跨线程访问）——不要为了"规范"硬加一层 handler，过度桥接。
- 纯计算、不碰 Document——后台线程随便跑。
- 只读查询——仍建议主线程执行，但不属于"启动事务"违规。

### 作者在书中警告的失败模式

- **线程外启动事务抛异常**——这是本反例的核心，任何绕行（Lock 包一下、async 包一下）都无效。
- **非模态对话框外启动事务抛异常**——只有模态对话框（ShowDialog）期间例外。
- **IUpdater Execute 内不能开新事务**（第5章 5.6.2）——即使在主线程的受支持工作流内，更新器也不能开新事务。
- **事件回调中事务行为受限**（第5章 5.2.2）。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：新版本 External Event 支持 `Raise` 失败原因诊断、以及 `ExternalEvent` 的重复触发行为更明确，修复此类 bug 的手段更多。
- 未覆盖 async/await 上下文捕获陷阱（`ConfigureAwait` 与否会影响回到哪个线程）——现代代码里这是高频踩点。
- 未讨论"后台线程里能否读模型"的边界——只读也不是完全免费，保守起见都回主线程。

### 容易混淆的邻近方法论

- "后台线程跑计算"（合法） vs "后台线程写模型"（非法）：区分点只有一个——是否触碰 Document/Transaction。
- 模态对话框（可开事务） vs 非模态对话框（不可开事务）：最容易踩的反差规则。

---

## 相关 skills

本 skill 与其他 skill 无明显依赖/对比/组合关系（独立性强）。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4（详见 test-prompts.json）
- **蒸馏时间**: 2026-08-27
