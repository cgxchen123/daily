---
title: "用 functools.partial 固定常用参数"
date: 2026-05-25
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 functools.partial 固定常用参数

partial 可以将常用参数预先绑定到函数，减少重复传参并让调用点更清晰。但它不能替代合理的接口设计；过度绑定会隐藏依赖，调试时也需要确认实际参数来源。

## 实施方法

先定义成功条件和失败时的行为，把事实、假设与待验证问题分开记录。对关键操作明确权限、输入边界和恢复方式，并为验证结果保留版本、命令和可复查证据。涉及敏感信息时遵循最小化原则，不要把秘密复制到日志或示例。

## 示例

```python
from functools import partial
def multiply(a: int, b: int) -> int:
    return a * b
double = partial(multiply, 2)
assert double(4) == 8
print(double(7))
```

## 验收清单

至少覆盖正常路径、边界路径和失败路径；说明示例未覆盖的限制。生产环境需根据真实并发、权限和依赖条件补充测试，未实际执行的测试不得宣称通过。
