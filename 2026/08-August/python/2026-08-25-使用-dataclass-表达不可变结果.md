---
title: "使用 dataclass 表达不可变结果"
date: 2026-08-25
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 使用 dataclass 表达不可变结果

对只读结果对象使用冻结数据类可以降低意外修改的风险，并让字段结构更明确。不过冻结只限制属性重新赋值，不会自动让内部列表或字典不可变；需要深层不可变时，应选用元组或只读映射等结构。

## 实施流程

1. 定义输入、输出和可观察的成功条件。
2. 明确失败时的停止、重试或恢复方式。
3. 在低风险环境验证，再记录实际运行结果。
4. 将版本、依赖和未覆盖范围写入交付记录。

## 示例

```python
from dataclasses import dataclass
@dataclass(frozen=True)
class Result:
    name: str
    count: int
r = Result('items', 3)
assert r.count == 3
print(r)
```

## 验收与限制

至少覆盖正常、边界和失败路径；示例只说明核心机制，生产环境需要专项测试。未实际执行的测试不得宣称通过，涉及外部服务时还应记录其版本和实际行为。
