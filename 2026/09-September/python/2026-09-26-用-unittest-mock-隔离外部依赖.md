---
title: "用 unittest.mock 隔离外部依赖"
date: 2026-09-26
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 unittest.mock 隔离外部依赖

单元测试如果依赖真实网络、时间或第三方服务，会变慢且难以重复。mock 可以模拟外部依赖的返回值和异常，但不应过度模拟内部实现，否则测试可能通过而真实集成失败。关键路径仍需集成测试。

## 实施步骤

1. 明确目标、输入边界和成功条件。
2. 在产生副作用前完成校验，并区分失败类型。
3. 保存版本、命令、结果和未覆盖范围。
4. 对发布或数据变更准备恢复方案，并在低风险环境验证。

## 示例

```python
from unittest.mock import Mock
client = Mock()
client.fetch.return_value = {'ok': True}
assert client.fetch()['ok'] is True
client.fetch.assert_called_once()
print('mock behavior verified')
```

## 验收清单

覆盖正常、边界和失败路径，检查文档与真实文件一致。示例仅展示核心机制，生产环境需要专项测试；未实际执行的测试不得宣称通过。
