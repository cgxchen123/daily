---
title: "评估 RAG 别只看几个案例：建立可复现的检索测试集"
date: 2026-10-19
category: trends
tags: [trends, rag, evaluation]
difficulty: intermediate
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 评估 RAG 别只看几个案例：建立可复现的检索测试集

## 真实痛点

只挑几个看起来成功的问答展示，无法发现检索排序变化带来的退化。

## 核心原理

固定查询、相关文档标注与排序结果，计算 Recall@K 和 MRR。指标只描述测试集，不等于真实用户体验。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
def recall_at_k(relevant: set[str], ranked: list[str], k: int) -> float:
    if k < 1 or not relevant:
        raise ValueError("k 和相关文档集合必须有效")
    return len(set(ranked[:k]) & relevant) / len(relevant)

def reciprocal_rank(relevant: set[str], ranked: list[str]) -> float:
    for rank, doc in enumerate(ranked, 1):
        if doc in relevant:
            return 1 / rank
    return 0.0

if __name__ == "__main__":
    relevant = {"doc-a", "doc-c"}
    ranked = ["doc-b", "doc-c", "doc-a"]
    print("Recall@2:", recall_at_k(relevant, ranked, 2))
    print("MRR:", reciprocal_rank(relevant, ranked))
~~~

## 运行与验证

保存为 rag_metrics.py 并运行。改变 ranked 顺序，观察指标如何变化。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

测试集标注不完整会扭曲指标；还应评估无答案查询、引用支持度、延迟与成本。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
