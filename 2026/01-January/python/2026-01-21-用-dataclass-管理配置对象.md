---
title: "用 dataclass 管理配置对象"
date: 2026-01-21
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 dataclass 管理配置对象

把相关配置放进明确的数据结构，能减少散落在函数参数里的隐式约定。使用数据类时，应明确必填字段、默认值和校验责任；不要把配置解析、环境变量读取和业务执行混在同一个构造过程里。示例中使用冻结数据类表示只读配置，并在入口校验范围。

## 示例与验证

保存示例并在对应环境中执行；Git 命令应在目标仓库目录执行。

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class RetryConfig:
    attempts: int
    delay_seconds: float

    def validate(self) -> None:
        if self.attempts < 1:
            raise ValueError("attempts must be >= 1")
        if self.delay_seconds < 0:
            raise ValueError("delay must be non-negative")

if __name__ == "__main__":
    config = RetryConfig(3, 0.5)
    config.validate()
    print(config)
```

## 验收与限制

验证正常输入、边界输入和失败路径；确认没有意外副作用，并记录真实运行环境与输出。示例不等于生产级实现，涉及并发、权限、网络或真实数据时应增加针对性测试。未实际执行的检查不得标记为通过。
