---
title: "用 heapq 处理 Top-K 问题"
date: 2026-05-01
category: python
tags: [python, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 heapq 处理 Top-K 问题

只需要前 K 个最大或最小元素时，完整排序可能做了不必要的工作。heapq 可以维护较小堆，在 K 远小于总数据量时更有优势；但若需要全部排序或 K 接近数据总量，应先比较实际复杂度和常数开销。

## 落地流程

1. 明确目标、输入来源和成功条件。
2. 识别异常、权限、并发与恢复边界。
3. 在低风险环境验证，再将结果纳入发布检查。
4. 记录实际版本、命令、结果与未覆盖范围。

## 示例

```python
import heapq
values = [9, 1, 7, 3, 8, 2]
assert heapq.nlargest(3, values) == [9, 8, 7]
print(heapq.nsmallest(2, values))
```

## 验收与限制

示例只说明核心机制，不能替代项目专项测试。至少覆盖正常、边界和失败路径；未实际执行的测试不得标记为通过。涉及生产数据或不可逆操作时，应先备份并确认回滚方案。
