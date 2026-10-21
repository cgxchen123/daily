---
title: "使用 functools.lru_cache 缓存纯函数"
date: 2026-08-17
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 使用 functools.lru_cache 缓存纯函数

缓存适合输入相同且结果稳定的纯函数，可减少重复计算。但缓存会占用内存，且当结果依赖时间、权限或外部状态时，缓存可能返回过期数据。应设定容量上限，并明确失效条件。

## 实施流程

1. 定义输入、输出和可观察的成功条件。
2. 明确失败时的停止、重试或恢复方式。
3. 在低风险环境验证，再记录实际运行结果。
4. 将版本、依赖和未覆盖范围写入交付记录。

## 示例

```python
from functools import lru_cache
@lru_cache(maxsize=128)
def square(n: int) -> int:
    return n * n
assert square(4) == 16
print(square.cache_info())
```

## 验收与限制

至少覆盖正常、边界和失败路径；示例只说明核心机制，生产环境需要专项测试。未实际执行的测试不得宣称通过，涉及外部服务时还应记录其版本和实际行为。
