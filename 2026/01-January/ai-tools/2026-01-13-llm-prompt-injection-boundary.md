---
title: "把提示注入当作输入边界问题，而不是提示词技巧"
date: 2026-01-13
category: ai-tools
tags: [llm, prompt-injection, security]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 把提示注入当作输入边界问题，而不是提示词技巧

当大模型读取网页、邮件或文档时，内容里可能包含试图改变模型行为的指令。系统提示词可以声明规则，却不能保证模型总会忽略不可信内容。因此，不能仅靠一句“忽略恶意指令”保护工具和数据。

## 把数据与权限分开

推荐流程是：模型读取内容并提出候选动作，独立策略层验证动作和参数，再由执行器按最小权限执行。文档中的文字不应自动获得调用工具、发送消息或读取其他文件的权限。

## 一个简单的策略门

```python
ALLOWED_TOOLS = {"search_docs", "summarize_text"}


def authorize(tool_name: str, args: dict) -> None:
    if tool_name not in ALLOWED_TOOLS:
        raise PermissionError("tool is not allowed")
    if not isinstance(args, dict):
        raise ValueError("arguments must be an object")
    if tool_name == "search_docs":
        query = args.get("query")
        if not isinstance(query, str) or not query.strip():
            raise ValueError("query must be a non-empty string")


if __name__ == "__main__":
    authorize("search_docs", {"query": "deployment guide"})
    try:
        authorize("send_email", {"to": "someone@example.com"})
    except PermissionError:
        print("blocked unknown tool")
```

这段代码只演示白名单和参数校验，并不构成完整的 Agent 安全系统。生产环境还要做数据访问控制、审批、审计和执行隔离。

## 实施清单

- 将外部文档标记为不可信数据，而非系统指令。
- 对每个工具定义独立的输入 schema 和权限。
- 高影响动作默认需要人工批准。
- 限制模型能访问的数据范围，不要把所有密钥放入上下文。
- 记录拒绝原因，并测试恶意指令混在正常文本中的情况。

提示注入不是靠更强硬的措辞就能彻底解决。核心是即使模型被误导，权限边界仍然能阻止未授权动作。
