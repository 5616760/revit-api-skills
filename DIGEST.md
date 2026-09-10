# 《API开发指南 Autodesk Revit》— 蒸馏精华 DIGEST

> 宦国胜主编（中国水利水电出版社，2016.12）。经仓颉蒸馏流水线产出 **159 个可调用技能**，覆盖 Revit API 二次开发八大域。本文是全书地图 + 使用手册。

## 一、这套技能是什么

不是读书笔记，而是 **159 个"遇到什么开发问题 → 调哪个技能"的可执行经验包**。每个技能 = SKILL.md，含六段：
- **R** 原文摘录（书页出处）｜**I** 思维模型｜**A1** 书中案例｜**A2** 触发场景｜**E** 可执行步骤｜**B** 边界

已通过红线质检（description ≤300 字、引文 ≤150 字）、200 条知识关联（depends-on / contrasts-with / composes-with）、结构全绿与独立盲测（**94% 调度通过率**，诱饵 100%）。

## 二、全书地图（19 主题组）

| 组 | 主题 | 组 | 主题 |
|---|---|---|---|
| A | 插件架构与命令入口 | J | 参数与数据存储 |
| B | 事务与重生成 | K | 事件与更新器（DMU） |
| C | 图元检索与过滤 | L | 故障处理 |
| D | 文档导航与图元基础 | M | 性能与点云 |
| E | 视图 | N | 工作共享 |
| F | 族 | O | 链接与外部引用 |
| G | 几何 | P | 导出 |
| H1 | 建筑构件（墙/板/洞口/轴网） | Q | UI（功能区与对话框） |
| H2 | 空间系统与分析模型 | R | 参数细节（单位与存储） |
| I | MEP（水暖电） | | |

**核心概念链路**：插件入口(A) → 事务(B) → 检索(C/D) → 操作(F/G/H) → 数据(J/R) → 失败处理(L) → 异步与自动化(K) → 协同与交付(N/O/P)。

## 三、推荐学习顺序（8 阶段）

1. **地基**：对象模型与插件入口（A、D 起步：revit-category-family-symbol-instance、revit-plugin-entry-types）
2. **改模型前提**：事务三件套（B：transaction-hierarchy、regenerate、failure-options）
3. **检索与过滤**（C：collector、filter、boundingbox）
4. **图元操作与族**（F、G、H）
5. **参数与数据存储**（J、R）
6. **故障处理**（L）
7. **事件、更新器与命令集成**（K、Q）
8. **协同交付**（N、O、P）

对应知识点间的依赖/对比/组成关系见 INDEX.md 引用图；术语查 GLOSSARY.md（110 词条）。

## 四、如何使用（最重要）

**使用方式 = 用自然语言提问，让 AI 自动选中对应技能。** 不必记 159 个名字，只需描述你的场景：

### 4.1 提问模板
> 描述「我正在做什么 + 卡在哪 + 期望结果」，尽量带类名/API 名。

- ✅ 好例子："用 NewFamilyInstance 放门失败，是不是 FamilySymbol 要先 Activate？"
- ❌ 差例子："帮我写 Revit 插件"（太宽，AI 会拆成多步再分别命中）

### 4.2 技能自动触发的判断信号
每个技能 frontmatter 的 `description` 就是它的"触发面"。AI 看到你的问题与某 description 匹配即自动加载该 SKILL.md 执行。遇到以下情况分别会命中：
- **决策类**（"该用 A 还是 B"）→ I/模型 + E/步骤 类技能
- **报错类**（"编译不过/抛异常/没效果"）→ 反例技能（name 含 failure/counter-example/absence）
- **查询类**（"怎么拿到 XX"）→ 导航/获取类技能

### 4.3 配套查阅
- 不知道问哪个 → 先查 `INDEX.md`（按主题浏览 159 个技能名与一句话说明）
- 想学完整脉络 → 按第 3 节 8 阶段顺序逐技能读 SKILL.md
- 查术语/类 → `GLOSSARY.md`

### 4.4 质量与边界
- 每技能含 `test-prompts.json`（真实触发测试，3-5 正例 + 诱饵 + 边界）与 `test-results.md`（测试报告），可据此验证 AI 是否选对。
- **适用边界**：内容基于 2016 版 Revit（2016/2017 API 时代），新版 API 若报"已过时/改名"，请以官方文档为准，本库用于方法论与惯性陷阱识别。

## 五、产出位置

- 技能集安装于：`~/.workbuddy/skills/revit-*`（159 个，全局可用）
- 源工程：`G:\DATA\WorkBuddy\Skills\MySkill\RevitAPI\`（含 BOOK_OVERVIEW.md、INDEX.md、GLOSSARY.md、_stage3/ 关系网络、_stage4/ 测试与判卷证据、PIPELINE_STATE.md 全流程记录）
