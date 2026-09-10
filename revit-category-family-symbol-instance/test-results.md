# 测试结果 — revit-category-family-symbol-instance

- **测试方式**: 独立 sub-agent 盲测（pack 抽样，隐藏 type/expected_behavior/notes，基于 159 skill 注册表做激活选择题）
- **测试时间**: 2026-08-30

| 类型 | 通过/总数 |
|---|---|
| should_trigger | 3/4 |
| should_not_trigger | 2/3 |
| edge_case | 1/2 |

**总通过率: 6/9 (67%)**

## 失败 case
- `should-trigger-05` (MISSING): 盲测结果缺失
- `should-not-trigger-03` (MISSING): 盲测结果缺失
- `edge-case-02` (edge_case): chosen=revit-system-vs-component-family | expected: 激活本 skill 应答：四层模型对系统族与构件族都成立，但入口不同——系统族的 Symbol/Family 挂在文档类型集合上，构件族走 LoadFamily
