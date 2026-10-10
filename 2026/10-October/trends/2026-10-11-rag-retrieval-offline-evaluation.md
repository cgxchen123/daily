---
title: "RAG 检索效果别靠感觉：用 Recall@K 和 MRR 做离线回归"
date: 2026-10-11
category: trends
tags: [rag, information-retrieval, evaluation, recall-at-k, mrr, python]
difficulty: beginner
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# RAG 检索效果别靠感觉：用 Recall@K 和 MRR 做离线回归

## 真实痛点：回答变差，不一定是模型变差

给知识库问答系统换了切分方式、嵌入模型或重排策略后，少数问题突然答不准，很容易只盯着最终答案调提示词。但问题可能更早发生：正确文档根本没有进入检索结果，或者它被排在很后面。没有一组固定问题和人工标注的相关文档，每次改动只能凭几个例子判断，回归也难以复现。

本文做一个本地离线评测器：把查询、应该命中的文档 ID、系统返回的排序结果写成 JSONL，再计算 Recall@K 与 MRR@K。不需要模型 API、向量数据库或第三方 Python 包。

## 核心原理：两个指标看不同问题

- **Recall@K**：前 K 个结果覆盖了多少个已标注的相关文档。它关注“该找回来的有没有找回来”。
- **MRR@K**：在前 K 个结果中，第一个相关文档排名的倒数；没有命中就记 0。它关注“第一个有用结果排得够不够靠前”。

程序先按每条查询分别计算，再对查询等权平均，避免相关文档较多的某一条查询自动主导总分。这与信息检索评测中按主题汇总指标的常见做法一致。指标定义可参考 [NIST TREC 评测说明](https://trec.nist.gov/pubs/trec30/papers/Overview-2021.pdf) 和 [Elasticsearch 排名评估文档](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/search-rank-eval)。

比如一条查询标注了两个相关文档，系统只在前三名找回其中一个，那么 Recall@3 是 0.5；如果它排在第二位，该查询的 Reciprocal Rank 就是 1/2。没有相关结果时，两项得分都可能是 0。这里的“相关”必须由任务标注定义，不能简单把关键词相同当成相关，也不要在看到系统输出后再倒过来修改标签。

## 完整可运行代码

保存为 `eval_retrieval.py`。输入文件每行一个 JSON 对象，必须有 `query`、`relevant` 和 `ranked` 三个字段；其中 `relevant` 是人工标注的相关文档 ID 列表，`ranked` 是系统按相关性从高到低返回的 ID 列表。ID 必须是非空字符串，列表中不能重复。

```python
from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path


def positive_int(value: str) -> int:
    number = int(value)
    if number <= 0:
        raise argparse.ArgumentTypeError("K 必须是正整数")
    return number


def validate_ids(value: object, field: str, line_number: int, allow_empty: bool) -> list[str]:
    if not isinstance(value, list):
        raise ValueError(f"第 {line_number} 行的 {field} 必须是数组")
    if not allow_empty and not value:
        raise ValueError(f"第 {line_number} 行的 {field} 不能为空")
    if any(not isinstance(item, str) or not item.strip() for item in value):
        raise ValueError(f"第 {line_number} 行的 {field} 必须只包含非空字符串")
    if len(value) != len(set(value)):
        raise ValueError(f"第 {line_number} 行的 {field} 存在重复 ID")
    return value


def load_cases(path: Path) -> list[dict[str, object]]:
    cases: list[dict[str, object]] = []
    with path.open("r", encoding="utf-8") as file:
        for line_number, raw_line in enumerate(file, start=1):
            if not raw_line.strip():
                continue
            try:
                record = json.loads(raw_line)
            except json.JSONDecodeError as exc:
                raise ValueError(f"第 {line_number} 行不是有效 JSON：{exc.msg}") from exc
            if not isinstance(record, dict):
                raise ValueError(f"第 {line_number} 行必须是 JSON 对象")
            if not isinstance(record.get("query"), str) or not record["query"].strip():
                raise ValueError(f"第 {line_number} 行的 query 必须是非空字符串")
            relevant = validate_ids(record.get("relevant"), "relevant", line_number, False)
            ranked = validate_ids(record.get("ranked"), "ranked", line_number, True)
            cases.append({"query": record["query"], "relevant": relevant, "ranked": ranked})
    if not cases:
        raise ValueError("测试集没有有效记录")
    return cases


def evaluate(cases: list[dict[str, object]], k: int) -> tuple[float, float]:
    recall_scores: list[float] = []
    reciprocal_ranks: list[float] = []
    for case in cases:
        relevant = set(case["relevant"])
        top_k = case["ranked"][:k]
        recall_scores.append(len(relevant.intersection(top_k)) / len(relevant))
        reciprocal_rank = 0.0
        for rank, document_id in enumerate(top_k, start=1):
            if document_id in relevant:
                reciprocal_rank = 1.0 / rank
                break
        reciprocal_ranks.append(reciprocal_rank)
    return sum(recall_scores) / len(cases), sum(reciprocal_ranks) / len(cases)


def main() -> int:
    parser = argparse.ArgumentParser(description="计算检索测试集的 Recall@K 和 MRR@K")
    parser.add_argument("dataset", type=Path, help="每行一个 JSON 对象的 JSONL 文件")
    parser.add_argument("--k", type=positive_int, default=5, help="评估前 K 个结果，默认 5")
    args = parser.parse_args()

    try:
        cases = load_cases(args.dataset)
    except (OSError, ValueError) as exc:
        print(f"错误：{exc}", file=sys.stderr)
        return 2

    recall, mrr = evaluate(cases, args.k)
    print(f"Queries: {len(cases)}")
    print(f"Recall@{args.k}: {recall:.3f}")
    print(f"MRR@{args.k}: {mrr:.3f}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## 立即运行

创建一个三条记录的样例 `eval.jsonl`：

```jsonl
{"query":"读取 CSV","relevant":["csv-doc","path-doc"],"ranked":["python-intro","csv-doc","other-doc"]}
{"query":"文件路径","relevant":["path-doc"],"ranked":["path-doc","io-doc","os-doc"]}
{"query":"字典排序","relevant":["dict-sort"],"ranked":["list-sort","tuple-sort","dict-sort"]}
```

运行命令：

```bash
python eval_retrieval.py eval.jsonl --k 3
```

上面样例的结果应为 `Queries: 3`、`Recall@3: 0.833`、`MRR@3: 0.611`。正式评测时，把 `ranked` 替换成当前检索器真实返回的结果，而不是手工编写一份“看起来正确”的排名。

## 工程思考与升级方向

离线评测的可信度取决于标注集。先固定一批真实用户问题，人工标出相关文档，并保留数据版本；比较不同检索策略时，保持查询和标注不变。每次升级切分、嵌入或重排配置后重新运行脚本，把指标变化作为回归信号，而不是只挑成功案例展示。

Recall 和 MRR 不评价生成答案是否准确，也不能证明文档内容最新、完整或安全。它们只能回答检索排序的一部分问题。下一步可以加入 Precision@K、nDCG@K、分问题类别的统计和置信区间；上线前还应检查答案引用是否真正支持结论。指标变好是调查线索，不是系统质量的全部结论。
