---
name: revit-new-opening-constraints
description: |
  NewOpening 抛异常时。最常见两个原因：竖井洞口 topLevel 必须高于 bottomLevel；墙洞口 pntStart/pntEnd 必须是能构成矩形的角点坐标。防御：Revit 创建 API 普遍内嵌隐式几何/上下文前置约束，违反即抛异常——创建前先显式验证几何关系与依赖存在性，而非依赖运行时报错。
  不适用：选重载走 revit-new-opening-overloads、读边界见 revit-opening-boundary-reading。
  Trigger："竖井洞口抛异常"、"创建洞口失败"、"两点不构成矩形"、"topLevel 不高于 bottomLevel"。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.1.7 洞口（p156）
tags: [opening, exception-conditions, argument-validation, shaft-opening, error-prevention]
related_skills:
  - slug: revit-new-opening-overloads
    relation: depends-on
---

# NewOpening 参数约束与异常条件

## R — 原文 (Reading)

> 新建洞口图元：用来在项目中创建竖井洞口。但是，必须确保顶层高于底层，否则将引发异常。在直墙或弧墙上创建洞口：用来在墙上创建矩形洞口。pntStart 和 pntEnd 坐标必须是能形成矩形的角点坐标。例如，矩形的左下角和右上角；否则将引发异常。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》 第3章 3.1.7 节洞口（p156）

---

## I — 方法论骨架 (Interpretation)

创建 API 抛异常，很多时候不是运气差，而是违反了 Revit 内嵌的隐式前置约束。NewOpening 的两个重载各带一组可验证的约束：

竖井洞口（双标高重载）：topLevel 必须高于 bottomLevel，标高序反了直接抛异常。墙洞口（两点矩形重载）：pntStart 与 pntEnd 必须是能构成矩形的角点——例如左下角与右上角；给任意两个点（比如对角方向不对）就抛异常。此外墙上只有矩形洞口这一种形态，非矩形需求不应走到这个 API（那是能力边界，不是参数校验问题）。

通用防御策略由此而来：Revit 创建 API 普遍内嵌"隐式几何/上下文前置约束"，违反即抛异常。所以正确姿势是"先验证，后调用"——调用任何创建方法前，先显式验证几何关系（顺序、共面、成矩形）与依赖存在性（类型已加载、标高存在），而不是把希望寄托在运行时的异常消息上。这个"前置能力约束"陷阱与族实例翻转（IsWorkPlaneFlipped 不可用时抛异常）、尺寸 Label 设置非法时抛异常同族。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 竖井洞口标高序（3.1.7）
- **问题**: 创建竖井洞口时抛异常。
- **方法论的使用**: 检查 topLevel 与 bottomLevel 的顺序。
- **结论**: 顶层必须高于底层是隐式约束。
- **结果**: 顺序反转即抛异常。

### 案例 2: 墙洞口矩形角点（3.1.7）
- **问题**: 墙洞口创建失败。
- **方法论的使用**: 验证两点能构成矩形角点（左下+右上）。
- **结论**: 角点约束可事先验证。
- **结果**: 非角点组合抛异常。

### 案例 3: 前置约束陷阱的同族复现（3.2.3、3.3.2）
- **问题**: 其他创建场景同样踩"前置能力约束"。
- **方法论的使用**: 对不允许工作平面翻转的实例设置 IsWorkPlaneFlipped 抛异常；对无法标注的尺寸设 Label 抛非法操作异常。
- **结论**: "先验证后调用"是通用防御策略。
- **结果**: 见 revit-instance-flip-state-check、s2b-c08。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. NewOpening 抛异常，需要定位根因。
2. 写创建洞口的代码，想提前避开异常。
3. 写通用创建工具，需要"先验证后调用"的检查清单。
4. 怀疑墙上非矩形洞口需求被误当成参数问题。

### 语言信号 (用户的话里出现这些就应激活)

- "创建竖井洞口抛异常" / "shaft opening throws an exception"
- "墙洞口创建失败" / "wall opening creation failed"
- "topLevel 必须高于 bottomLevel" / "top level must be above bottom level"
- "两点不是矩形角点" / "points do not form a rectangle"

### 与相邻 skill 的区分

- 与 `revit-new-opening-overloads`：本 skill 在重载选定后验证参数与异常，依赖其重载选型结果。
## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **定位异常重载**
   - 完成标准: 明确抛异常的 NewOpening 属于哪个重载（竖井/墙/板/族实例）。

2. **逐项验证前置约束**
   - 完成标准: 竖井重载验证 topLevel > bottomLevel；墙重载验证两点构成矩形（如左下+右上），否则先修正坐标；族实例重载验证 CurveArray 闭合、eRefFace 有效。
   - 判停条件: 若需求是"墙上非矩形洞口"，判停——不是参数问题，走编辑墙轮廓路径。

3. **先验证后调用**
   - 完成标准: 修改代码为"调用前显式验证几何关系与依赖存在性"；创建后验证洞口非 null、位置正确；异常不再出现。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 选择 NewOpening 重载本身：那是 revit-new-opening-overloads。
- 读取已有洞口边界：那是 revit-opening-boundary-reading。

### 作者在书中警告的失败模式

- 竖井标高序反转：topLevel 不高于 bottomLevel 即抛异常。
- 墙洞口两点不成矩形：不是任意两点都能建墙洞。
- 墙上非矩形洞口：不能靠改参数绕过，API 不支持（能力边界）。

### 作者的盲点 / 时代局限

- 书基于 Revit 2014：约束清单在后续版本保持；新版本对墙洞口新增了轮廓编辑能力，但 NewOpening 的矩形约束不变。Generic"先验证后调用"的防御策略同样适用其他创建 API。

### 容易混淆的邻近方法论

- 参数校验（几何前置约束）vs 运行时异常兜底：前者主动预防，后者被动补救；本 skill 强调前者。
- 矩形角点约束与"矩形的两对对角点"：角点必须是对角关系（左下+右上），不是任意两点。

---

## 相关 skills

- **revit-new-opening-overloads**（depends-on）：本 skill 在重载选定后验证参数与异常，依赖其重载选型结果。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
