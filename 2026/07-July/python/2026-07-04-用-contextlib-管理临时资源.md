---
title: "用 contextlib 管理临时资源"
date: 2026-07-04
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 contextlib 管理临时资源

临时资源需要在异常路径中也得到清理。上下文管理器能集中管理生命周期，但清理逻辑应简单、可靠，避免掩盖原始异常。对于文件、锁和网络连接，还应考虑进程被强制终止时的恢复行为。

## 实施步骤

1. 明确数据来源、目标范围和成功条件。
2. 在产生副作用前完成输入与权限检查。
3. 对失败、重试和恢复路径做独立验证。
4. 保存版本、命令和结果，避免把假设当作验证事实。

## 示例

```python
from contextlib import contextmanager
@contextmanager
def resource():
    print('open')
    try: yield
    finally: print('close')
with resource():
    print('work')
```

## 验收与限制

至少覆盖正常路径、边界条件和失败路径。示例只展示核心机制，生产系统还需根据真实环境补充测试；未执行的测试不得宣称通过。
