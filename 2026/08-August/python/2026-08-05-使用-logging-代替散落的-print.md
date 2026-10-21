---
title: "使用 logging 代替散落的 print"
date: 2026-08-05
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 使用 logging 代替散落的 print

print 适合短小的一次性脚本，但服务和自动化任务通常需要日志级别、时间、模块名和异常堆栈。日志应保持结构化并支持关联请求；不要在每个模块重复配置全局处理器。

## 实施步骤

1. 定义可观察的成功条件和失败状态。
2. 在产生副作用前完成输入与状态校验。
3. 记录版本、运行条件和验证证据。
4. 对不可逆操作准备备份、审批或恢复方式。

## 示例

```python
import logging
logging.basicConfig(level=logging.INFO, format='%(levelname)s %(message)s')
logger = logging.getLogger(__name__)
logger.info('job_started run_id=%s', 'demo-1')
```

## 验收与限制

覆盖正常路径、边界条件和失败路径；说明尚未验证的部分。示例仅展示核心机制，生产环境需要专项测试。未实际运行的测试不得宣称通过。
