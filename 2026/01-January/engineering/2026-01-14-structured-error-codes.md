---
title: "给程序错误定义稳定的错误码，别让调用方解析文案"
date: 2026-01-14
category: engineering
tags: [api, error-handling, observability]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 给程序错误定义稳定的错误码，别让调用方解析文案

如果调用方根据错误提示中的某段中文判断发生了什么，文案一改就可能导致集成失败。面向程序的接口应提供稳定错误码，面向人的说明可以改写、翻译或补充上下文。

## 一个小型错误结构

```python
from dataclasses import dataclass


@dataclass
class AppError(Exception):
    code: str
    message: str
    retryable: bool = False


def load_record(record_id: str) -> dict:
    if not record_id.strip():
        raise AppError("INVALID_ID", "record id is required")
    raise AppError("NOT_FOUND", "record does not exist")


if __name__ == "__main__":
    try:
        load_record("")
    except AppError as exc:
        print(exc.code, exc.message, exc.retryable)
```

保存并执行后，调用方可以稳定判断 `INVALID_ID`，而不必比较整句文案。示例为了演示只覆盖错误结构；实际服务应让异常继承体系和 API 响应格式保持一致。

## 错误码的设计约束

- 一个错误码对应明确、可区分的情况。
- 不要把数据库 SQL、令牌或内部路径放进公开错误消息。
- 是否可重试应基于错误语义，而不是所有 5xx 都无限重试。
- 为错误码维护文档，并避免在不兼容变更中复用旧码。
- 日志中的 request ID 应能关联同一次请求，但不能作为认证凭据。

稳定错误契约能降低跨服务耦合，也让监控统计更可靠。文案服务于人，错误码服务于程序，两者不应互相替代。
