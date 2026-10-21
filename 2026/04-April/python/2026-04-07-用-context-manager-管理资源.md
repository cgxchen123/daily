---
title: "用 context manager 管理资源"
date: 2026-04-07
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 context manager 管理资源

文件、锁和连接等资源需要在成功与异常路径都正确释放。上下文管理器让清理逻辑更集中；但资源释放本身也可能失败，应避免让清理异常掩盖最初的业务异常，并明确事务提交与回滚语义。

## 推荐流程

先写明目的、输入来源、成功条件和失败时的动作。把可逆步骤与不可逆步骤分开，先在低风险环境验证；涉及外部依赖时，记录具体版本、权限和运行条件，避免根据默认行为推断安全性或可靠性。

## 示例与检查

```python
from pathlib import Path
from tempfile import TemporaryDirectory
with TemporaryDirectory() as tmp:
    p = Path(tmp) / 'demo.txt'
    p.write_text('sample', encoding='utf-8')
    assert p.read_text(encoding='utf-8') == 'sample'
print('temporary resource check passed')
```

## 验收清单

覆盖正常路径、边界条件和失败路径；记录实际执行命令、输出和未覆盖限制。示例仅展示核心机制，不能替代项目专项测试。未实际执行的测试不得宣称通过。
