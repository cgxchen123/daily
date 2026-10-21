---
title: "评估 AI 功能时先建立非 AI 基线"
date: 2026-01-20
category: trends
tags: [ai, evaluation, product-metrics]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 评估 AI 功能时先建立非 AI 基线

在产品中加入大模型功能后，演示效果往往很直观，但“看起来更聪明”不等于实际改善了用户任务。若没有非 AI 基线，团队很难判断收益来自模型、界面变化，还是测试样本本身更简单。

## 先定义任务和指标

为具体场景建立一组代表性输入，并定义可观察的成功标准。例如客服草稿助手可以记录：事实错误率、人工修改幅度、完成时间、拒绝率和用户满意度。不要只报告一个总体准确率，因为不同失败类型的成本可能不同。

## 用固定数据集做最小对照

```python
def improvement(baseline: float, candidate: float) -> float:
    if baseline < 0 or candidate < 0:
        raise ValueError("scores must be non-negative")
    return candidate - baseline


if __name__ == "__main__":
    delta = improvement(0.72, 0.81)
    assert round(delta, 2) == 0.09
    print(f"absolute improvement={delta:.2f}")
```

这里的数字只是演示计算方法，不代表任何真实模型或产品表现。真实评估必须保存样本来源、版本、提示词、运行配置和评分规则。

## 避免常见误判

- 同一批样本同时用于调参和最终报告，会高估效果。
- 只挑成功案例会掩盖失败分布。
- 模型版本、检索索引和系统提示变化都可能影响结果。
- 人工评分应有清晰准则，重要场景考虑双人复核。
- 评估质量、延迟、成本和安全风险，而不只是答案流畅度。

AI 功能是否值得上线，应由明确任务上的可重复证据决定。先与简单规则、人工流程或旧系统对照，才能知道模型究竟带来了什么增益。
