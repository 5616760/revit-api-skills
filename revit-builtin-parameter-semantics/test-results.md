# 测试结果 — revit-builtin-parameter-semantics

- **测试方式**: 独立 sub-agent 盲测（pack 抽样，隐藏 type/expected_behavior/notes，基于 159 skill 注册表做激活选择题）
- **测试时间**: 2026-08-30

| 类型 | 通过/总数 |
|---|---|
| should_trigger | 4/4 |
| should_not_trigger | 2/2 |
| edge_case | 1/2 |

**总通过率: 7/8 (88%)**

## 失败 case
- `edge-case-02` (edge_case): chosen=revit-parameter-index-lookup | expected: 明确 BuiltInParameter 枚举只覆盖 Revit 预定义内建参数，自定义共享参数不在枚举内；共享参数的稳定标识是其定义级 GUID，应走 GUID
