---
title: "用 functools.lru_cache 缓存纯函数"
date: 2026-04-23
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 functools.lru_cache 缓存纯函数

缓存适合输入相同且结果稳定的纯函数，可减少重复计算。但缓存会占用内存，且当结果依赖时间、权限或外部状态时，缓存可能返回过期数据。应设定最大容量，并明确缓存失效条件。

## 落地方法

先明确目标、输入来源和失败时的处理方式。把可验证条件写成检查项，分开记录事实、假设与待验证内容。涉及外部系统时应记录版本、时间和错误分类，避免保存凭据或不必要的个人数据。

## 示例

```python
from functools import lru_cache
@lru_cache(maxsize=128)
def square(n: int) -> int:
    return n * n
assert square(4) == 16
assert square.cache_info().hits >= 0
print(square.cache_info())
```

## 验收清单

覆盖正常路径、边界条件和失败路径；确认重复执行不会造成意外副作用。示例只展示核心机制，不能替代项目专项测试。未实际执行的测试不得宣称通过。
