---
name: revit-execute-parameter-semantics
description: |
  当写 Execute 方法想知道三参数分工、或命令失败时要高亮问题图元时调用。commandData=输入中枢（取 Application/UIDocument/视图）；message=错误输出（Fa
  iled/Cancelled 时弹对话框）；elements=失败图元集合（message 非空时高亮）。不适用：模型参数读取。trigger：commandData、message elements
   参数、高亮图元、execute parameters、highlight elements。

source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: p032-033
tags: [external-command, execute-signature, error-feedback, uidocument]
related_skills:
  - slug: revit-command-result-undo
    relation: composes-with
---

# 外部命令参数语义：commandData / message / elements

## R — 原文 (Reading)

> commandData 包含引用的应用程序和外部命令所需视图，从此参数直接或间接检索所有 Revit 数据。message 是输出参数，用于返回错误信息。elements 在返回 Failed 或 Cancelled 且消息不为空时，将突出显示在屏幕上。
>
> — 宦国胜, 第1章 1.3.2（约 p032–p033）

---

## I — 方法论骨架 (Interpretation)

`Execute` 的三个参数构成 Revit 特有的"输入—消息—高亮"三通道：

1. **commandData（输入中枢）**：一个 `ExternalCommandData` 对象，内含 `Application`（UIApplication，进而可拿 ActiveUIDocument/Document）和当前视图信息。**所有** Revit 数据的检索都必须从这个对象出发——它是命令与世界之间唯一的入口管道。
2. **message（错误输出）**：`ref string`，是个输出通道。命令失败时把一句话错误写进去，Revit 在返回 `Failed`/`Cancelled` 时把它弹成错误/警告对话框。
3. **elements（高亮集合）**：`ElementSet`，存放"罪魁祸首"图元。返回 `Failed`/`Cancelled` 且 message 非空时，这些图元会在屏幕上高亮，帮用户一眼定位问题对象。

用法套路：出错时 `message = "xxx失败"`，把失败相关的墙/族实例 Add 进 elements，`return Result.Failed`——用户看到错误文案 + 视图中亮起的图元。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 代码 1-4 用 message 返回异常
- **问题**: Execute 内捕获到异常，如何让 Revit 弹给用户？
- **方法论的使用**: catch 中 `message = ex.Message`，返回 `Result.Failed`。
- **结论**: message 是命令级错误文本通道。
- **结果**: Revit 显示含异常信息的错误对话框。

### 案例 2: commandData 导航检索（1.2.5）
- **问题**: 命令要拿到当前文档和选中图元。
- **方法论的使用**: `commandData.Application.ActiveUIDocument.Document` / 通过 UIDocument 取 Selection。
- **结论**: commandData 是检索一切数据的起点。
- **结果**: 检索演练代码从 commandData 逐层导航成功。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写 Execute 方法时不知道三个参数怎么用、ref 关键字忘了写导致编译失败。
2. 命令失败时想让 Revit 自动高亮"哪几个图元出了问题"，而不是干巴巴报错。
3. 想知道"commandData.Application 和启动时的 UIControlledApplication 有什么区别"。
4. 想区分"错误对话框（Failed）"和"警告对话框（Cancelled）"两种反馈。

### 语言信号 (用户的话里出现这些就应激活)

- "Execute 的三个参数 / commandData 是什么"
- "怎么让 Revit 高亮失败的图元"
- "message 参数怎么用 / elements 集合"
- "返回 Failed 时怎么显示错误"
- "execute parameters / commandData / highlight elements / error message dialog"

### 与相邻 skill 的区分

- 与 `revit-command-result-undo` 的区别: 本 skill 是三参数语义（尤其 message/elements 的反馈通道），result-undo 讲返回值如何驱动自动回滚。
- 与 `revit-execute-try-catch-message` 的区别: 本 skill 是参数分工全景，try-catch-message 专注异常如何填充 message。
- 与批次7参数专题的区别: 本 skill 的"参数"指方法参数，参数专题指模型参数（Parameter 对象）。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **确认 Execute 签名与三参数角色**
   - 完成标准: 签名含 `ExternalCommandData commandData`、`ref string message`、`ElementSet elements`，且能说出三者的分工。若签名不匹配，修正 ref/类型。

2. **补全错误反馈路径**
   - 完成标准: 所有失败路径都设置了 message（一句话可读文案），并把问题图元 `elements.Insert(元素)`；返回 Failed/Cancelled 时二者配套。成功路径返回 Succeeded 且不依赖 message。
   - 判停条件: 若用户只是想理解三参数语义、不需要改代码，完成步骤 1 即可停，输出“输入-消息-高亮”速查给用户。

3. **验证高亮与对话框行为**
   - 完成标准: 构造失败输入运行命令，确认：Revit 弹出含 message 内容的对话框 + 视图中 elements 图元被高亮。给用户提醒：elements 仅当 message 非空才高亮，且返回 Cancelled 弹的是 Warning 而非 Error。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 参数读取（Parameter 对象 / StorageType）——那是批次7参数专题。
- 事务内部逻辑——那是事务专题。

### 作者在书中警告的失败模式

- `message` 忘记用 `ref` → 赋值传不回调用方，Revit 显示空错误。
- 返回 `Failed` 却不设 message → 无提示的失败，用户不知为何失败。
- 返回 `Cancelled` 却被当成功处理——Cancelled 同样触发回滚（见 result-undo）。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014：新版本 `ElementSet` 已被 `ISet<ElementId>` 更广泛取代，但 Execute 签名保持向后兼容。
- 书未讲 message 长度/本地化建议；多语言环境下应避免硬编码错误文案。

### 容易混淆的邻近方法论

- `commandData.Application`（UIApplication，命令期）vs `UIControlledApplication`（启动期）——两者名字只差 Controlled，但能力范围完全不同。
- `message`（用户可读）vs `ex.StackTrace`（开发者诊断）——前者进对话框，后者进日志。

---

## 相关 skills

- revit-command-result-undo：composes-with——本 skill 讲三参数语义与反馈通道，result-undo 讲返回值如何驱动自动回滚，构成 Execute 结果处理的完整链路。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
