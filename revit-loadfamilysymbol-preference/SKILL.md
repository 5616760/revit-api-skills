---
name: revit-loadfamilysymbol-preference
description: |
  用户要按需加载单个族符号以省内存、LoadFamily 第二次运行返回 false、需要定位 .rfa 文件路径、或性能敏感的单类型放置时调用。不适用于：需要族内全部类型实例化、类型名不确定需先遍历的场景。关键 trigger："LoadFamily 返回 false"、"加载单个符号 vs 整个族"、"load family symbol"、"GetLibraryPaths"、"性能优化 加载族"。核心：单类型放置优先 LoadFamilySymbol（Name 须与 FamilySymbol.Name 完全一致）+ NewFamilyInstance；只有需要全部类型时才退回 LoadFamily。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.2.2 族（p159）
tags: [loadfamily, loadfamilysymbol, performance, family-loading, library-paths, revit-api]
related_skills:
  - slug: revit-family-document-edit-paths
    relation: contrasts-with
---

# 优先使用 LoadFamilySymbol 而非 LoadFamily 减少内存占用

## R — 原文 (Reading)

> 注意：若要提高应用程序的性能和减少内存使用，则应尽可能地加载特定的族符号而不是整个族对象。
>
> — 宦国胜, 《API开发指南 Autodesk Revit》第3章 3.2.2 族（p159）

---

## I — 方法论骨架 (Interpretation)

"按需加载"在 Revit 里落到一对方法：

- `LoadFamilySymbol(fileName, symbolName)`：**只加载族里的一个符号**（类型）。更快、更省内存，是单类型放置的默认选择。
- `LoadFamily(fileName)`：加载整个族（所有类型）。重，只在需要族内**全部类型**实例化时才用。

执行细节（全是坑）：

1. **Name 必须完全一致**：LoadFamilySymbol 的 symbolName 要与 `FamilySymbol.Name` 精确匹配，差一个字都加载不到。
2. **返回 false 语义**：`LoadFamily` 对已加载的族返回 **false（不是异常）**——可据此判断首次加载。
3. **定位 .rfa**：用 `GetLibraryPaths` 解析库路径，再拼文件名定位 .rfa。
4. **配合使用**：单类型放置的推荐组合 = `LoadFamilySymbol(...)` → `NewFamilyInstance(...)`。

反例对照：创建桌子案例需要全部类型时才用 LoadFamily；创建梁案例强调"加载单个族符号更快"。决策维度只有一个：**这次要用这个族的几种类型？**

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 创建梁用符号加载
- **问题**: 放一批梁，只需要一种型号。
- **方法论的使用**: 载入族符号而不是族，因为加载单个族符号更快。
- **结论**: 单类型场景，LoadFamilySymbol 是性能更优选择。
- **结果**: 加载更快、内存占用更低。

### 案例 2: 创建桌子需全部类型
- **问题**: 桌子要多种类型都用到。
- **方法论的使用**: 使用 LoadFamily 加载整个族，随后遍历符号。
- **结论**: 多类型需求时才值得 LoadFamily。
- **结果**: 全部类型可用，代价是更大加载开销。

### 案例 3: 二次加载判断
- **问题**: 代码第二次运行，LoadFamily 返回 false。
- **方法论的使用**: 识别返回 false = 已加载，据此跳过重复加载。
- **结论**: false 是状态信号而非错误。
- **结果**: 幂等加载逻辑成立。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写放置工具，只要放一种型号的构件。
2. 加载代码第二次运行返回 false，想知道为什么。
3. 性能敏感：批量放构件，想压缩加载开销。
4. 需要从库路径定位 .rfa 文件。

### 语言信号 (用户的话里出现这些就应激活)

- "LoadFamily 返回 false"（"LoadFamily returns false"）
- "只加载一个类型/符号"（"load only one symbol / type"）
- "加载整个族 vs 单个符号"（"load family vs family symbol"）
- "GetLibraryPaths"
- "提升加载性能/省内存"（"improve loading performance"）

### 与相邻 skill 的区分

- 与 `revit-family-document-edit-paths`：本 skill 是一次性加载符号使用，后者是"编辑+回载"流程，用途相对立。
## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **问清类型数量**
   - 本次需要族内几种类型？1 种 → LoadFamilySymbol；多种 → LoadFamily。
   - 完成标准: 明确决策依据（类型数量）。

2. **执行加载**
   - LoadFamilySymbol(fileName, symbolName)：symbolName 与 FamilySymbol.Name **精确一致**。
   - 需要路径 → GetLibraryPaths 解析 + 拼文件名。
   - 已加载判断：检查返回 false / 抛异常（已加载），幂等处理。
   - 完成标准: 目标符号/族已加载，无重复加载。

3. **放置并验证**
   - 单类型 → NewFamilyInstance(符号) 放置。
   - 完成标准: 实例创建成功，符号可查询。
   - 判停条件: 若用户要批量放置多实例，继续走 `revit-newfamilyinstances-batch-create`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 需要族内全部类型实例化（LoadFamilySymbol 不够用）。
- 类型名不确定、要先遍历符号再决定。

### 作者在书中警告的失败模式

- symbolName 与 FamilySymbol.Name 不完全一致 → 加载失败/找不到。
- 反复 LoadFamily 加载整个族 → 内存与性能浪费（书中明确警告）。

### 作者的盲点 / 时代局限

- 基于 Revit 2014：后续版本加载 API 新增变体（按路径/按 GUID），但"按需加载符号"策略与性能动机不变。

### 容易混淆的邻近方法论

- LoadFamily 返回 false（已加载）与"加载失败"是两回事——前者是状态，后者才需要处理。
- LoadFamilySymbol 的 Name 参数是**符号名**不是族名——传错对象是常见错误。

---

## 相关 skills

- **revit-family-document-edit-paths**（contrasts-with）：本 skill 是一次性加载符号使用，后者是"编辑+回载"流程，用途相对立。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: {{%}} (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
