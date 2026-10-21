---
title: "日志需要排错信息，但不应该泄露密钥"
date: 2026-01-03
category: security
tags: [security, logging, secrets, python]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 日志需要排错信息，但不应该泄露密钥

排查故障需要日志，但把完整请求头、环境变量或配置对象直接打印出来，可能把访问令牌、密码和个人信息写进日志平台。日志通常会被更多人员读取，也可能长期保存或转发，因此“只在调试环境打印一下”并不是可靠的安全策略。

## 用字段白名单代替整对象输出

最稳妥的方式不是不断增加黑名单，而是只记录排错真正需要的字段：

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
logger = logging.getLogger("service")


def log_request(request_id: str, method: str, route: str, status: int) -> None:
    logger.info(
        "request_complete request_id=%s method=%s route=%s status=%d",
        request_id,
        method,
        route,
        status,
    )


if __name__ == "__main__":
    log_request("req-123", "GET", "/health", 200)
```

保存为 `safe_logging.py` 后执行 `python safe_logging.py`。示例刻意不接收请求头、Cookie、Authorization 或整个请求对象。

## 如果确实要记录结构化字段

- 维护明确的允许字段列表。
- 对邮箱、手机号等个人信息按业务需求脱敏。
- 对异常堆栈设定访问权限和保留期限。
- 禁止把环境变量全集、配置文件全文写入日志。
- 对日志输出做测试，检查常见秘密字段不会出现。

## 为什么简单正则不够

密钥格式并不统一，新的供应商也可能引入不同令牌格式。正则脱敏可以作为第二道防线，却不应成为唯一防线。更重要的是从数据源头减少采集，限制日志读取权限，并对异常导出行为进行审计。

日志的目标是帮助定位问题，而不是完整复刻请求内容。每多记录一个字段，都应该能解释它对诊断的必要性。
