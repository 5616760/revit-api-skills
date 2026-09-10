# 测试结果 — revit-checkout-group-propagation

- **测试方式**: 独立 sub-agent 盲测（pack 抽样，隐藏 type/expected_behavior/notes，基于 159 skill 注册表做激活选择题）
- **测试时间**: 2026-08-30

| 类型 | 通过/总数 |
|---|---|
| should_trigger | 4/4 |
| should_not_trigger | 3/3 |
| edge_case | 1/2 |

**总通过率: 8/9 (89%)**

## 失败 case
- `edge-case-02` (edge_case): chosen=revit-checkout-group-propagation | expected: 不应调用本 skill，理由：只读操作不涉及检出，没有所有权传递问题。
