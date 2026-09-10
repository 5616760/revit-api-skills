---
name: revit-family-document-edit-paths
description: |
  用户要新建族文件、在代码里直接编辑 .rfa、在项目里改族几何后项目未更新、或困惑 EditFamily/LoadFamily/NewFamilyDocument 三者何时用时调用。不适用于：只改类型参数（用 FamilyManager 即可）、编辑系统族（不可行）。关键 trigger："在项目里修改门的族几何"、"项目里改了族怎么不生效"、"EditFamily / LoadFamily"、"NewFamilyDocument"、"编辑 rfa 文件"、"IsFamilyDocument"。核心：项目文档与族文档是两种上下文；编辑走"副本+回载"往返协议；已有实例需显式改 Symbol 才换型。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.3 族文件（p172）
tags: [family-document, editfamily, loadfamily, newfamilydocument, document-context, revit-api]
related_skills:
  - slug: revit-system-vs-component-family
    relation: depends-on
---

# 族文件创建与编辑的四种路径决策

## R — 原文 (Reading)

> API 不可以对系统族进行编辑。如果由 IsFamilyDocument 属性判定某文件是族文件，可由 Document 类来修改该族文件并访问族类型和参数。要编辑已存在的族，可使用 EditFamily()，完成后使用 LoadFamily() 将族重新载回主控文件。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.3 族文件（p172）

---

## I — 方法论骨架 (Interpretation)

族文档与项目文档是**两种 Document 上下文**，想编辑族，先选路径：

| 场景 | 路径 | 入口 |
|---|---|---|
| 新建一个族文件 | ① 新建 | `Application.NewFamilyDocument()` |
| 直接编辑 .rfa（打开即族文档） | ② 直编 | 先判 `IsFamilyDocument`，再用 Document 改 |
| 在**项目文档**里改既有族 | ③ 往返 | `EditFamily()` 打开副本 → 修改 → `LoadFamily()` 回载 |
| 从实例访问其族参数 | ④ 溯源 | `Family.OwnerFamily` / 族文档内参数访问 |

三条铁律：

1. **系统族不可编辑**——没有族文件可编辑，只能改类型属性（先见 revit-system-vs-component-family）。
2. **编辑走副本+回载**：EditFamily 打开的是族编辑副本，改完**必须 LoadFamily 载回主控文档**才生效；且已有实例**不会自动切换**到新类型——需要显式改 `FamilyInstance.Symbol`。
3. **两种文档不混用**：项目文档上调用族文档专用 API 会失败。

核心心智模型：项目是"使用地"，族文件是"产地"——产地里改了货，必须重新"发货"（回载）并让买家换货（改 Symbol）才在项目里看到。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: EditFamily 打开族文档列符号
- **问题**: 在项目里要编辑一个族的几何/嵌套符号。
- **方法论的使用**: 走路径 ③：EditFamily() 打开族文档，在族文档里枚举内嵌符号。
- **结论**: EditFamily 是"项目上下文 → 族上下文"的入口。
- **结果**: 成功打开族文档并定位符号。

### 案例 2: 完整回写闭环
- **问题**: 想在项目里给族新增一个类型并让某个实例用上它。
- **方法论的使用**: EditFamily → FamilyManager.NewType → LoadFamily 回载 → 改实例 Symbol 指向新类型。
- **结论**: 路径 ③ 的端到端时序：编辑→回载→换型缺一不可。
- **结果**: 新类型在项目中可用且实例正确切换。

### 案例 3: 新建族文件
- **问题**: 需要一个全新的族。
- **方法论的使用**: Application.NewFamilyDocument() 创建，进入路径 ② 语境用 FamilyManager 建类型。
- **结论**: 新建族文件后就在族文档上下文中。
- **结果**: 族文件创建成功并可继续编辑。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 在项目文档里用 API 修改门的族几何，发现项目没更新。
2. 写"族库工具"，要新建/编辑 .rfa 文件。
3. 拿到一个实例，想访问/修改它的族参数。
4. 分不清 EditFamily / LoadFamily / NewFamilyDocument 各自用途。

### 语言信号 (用户的话里出现这些就应激活)

- "在项目里改族的几何，项目不更新"（"edit family geometry in project"）
- "EditFamily / LoadFamily" 成对出现
- "新建族文件"（"create a new family document"）
- "IsFamilyDocument 判断"
- "改完族怎么让项目里的实例生效"

### 与相邻 skill 的区分

- 与 `revit-system-vs-component-family`：本 skill 的编辑路径仅对构件族成立，依赖其先判别系统族/构件族，系统族直接短路。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **判断当前上下文与目标**
   - 是项目文档还是族文档（IsFamilyDocument）？目标是新建/编辑/还是访问参数？
   - 完成标准: 明确落在四路径中的哪一条。

2. **按路径执行**
   - 新建 → NewFamilyDocument()，随后在族文档上下文操作。
   - 项目里改既有族 → EditFamily() → 修改 → **LoadFamily() 回载** → 逐实例改 `Symbol`。
   - 完成标准: 回载已完成、受影响实例显式切换类型。

3. **验证生效**
   - 项目文档中抽查实例类型与几何已更新；系统族直接告知不可编辑并转类型参数路径。
   - 完成标准: 修改在项目中可见，无"改了不生效"的遗留。
   - 判停条件: 若用户只是加参数/建类型（不改几何），则无需 EditFamily，直接走 FamilyManager skill。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 只改类型参数值/新增类型 → FamilyManager 即可，不必开族文档副本。
- 目标是系统族（墙/楼板）→ 不存在族文件编辑，只能改类型属性。

### 作者在书中警告的失败模式

- EditFamily 后忘 LoadFamily → 所有修改对项目不可见（副本被丢弃）。
- 回载后已有实例不自动换型 → 必须显式改 FamilyInstance.Symbol。
- "API 不可以对系统族进行编辑"。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本族编辑器 API 有扩展（如族文档中更多创建方法），但"副本+回载"的往返协议延续至今。

### 容易混淆的邻近方法论

- LoadFamily（编辑后回载）与 LoadFamilySymbol（按需加载单个符号，见 revit-loadfamilysymbol-preference）是两个不同用途，别混。
- 项目文档 Document 与族文档 Document 是同名但不同上下文——跨上下文调 API 是常见错误源。

---

## 相关 skills

- **revit-system-vs-component-family**（depends-on）：本 skill 的编辑路径仅对构件族成立，依赖其先判别系统族/构件族，系统族直接短路。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
