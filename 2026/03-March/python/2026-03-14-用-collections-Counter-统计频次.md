---
title: "用 collections.Counter 统计频次"
date: 2026-03-14
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 collections.Counter 统计频次

统计元素频次时，Counter 比手动维护字典更直接，并提供 most_common 等常用操作。但输入规模很大时，仍需考虑内存占用；如果只关心有限类别，可以在读取时过滤或采用流式聚合。

## 实施方法

先明确要解决的问题、影响范围和失败时的处理方式。将可验证条件写成检查项，并把事实、假设和待验证内容分开记录。若依赖外部服务，应保留版本、请求时间和错误分类，但不要在日志中保存凭据或不必要的个人数据。

## 示例

```python
from collections import Counter

values = ["a", "b", "a", "c", "a", "b"]
counts = Counter(values)
assert counts["a"] == 3
assert counts.most_common(1) == [("a", 3)]
print(counts)
```

## 验收清单

- 覆盖正常路径和至少一个失败路径。
- 确认重复执行或并发时不会产生意外结果。
- 记录实际运行环境、命令和结果。
- 说明示例未覆盖的生产限制，未执行的测试不得宣称通过。
