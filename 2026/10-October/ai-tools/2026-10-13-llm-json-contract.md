---
title: "让模型输出 JSON 更可靠：把解析、校验和重试分开"
date: 2026-10-13
category: ai-tools
tags: [ai-tools, json, validation]
difficulty: intermediate
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 让模型输出 JSON 更可靠：把解析、校验和重试分开

## 真实痛点

提示模型只输出 JSON，仍可能收到围栏、缺字段、错误类型或越界值。

## 核心原理

将语法解析、字段结构和业务约束分层处理；模型输出只能成为候选输入，不能替代授权判断。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import json

def parse_task(raw: str) -> dict:
    value = json.loads(raw)
    if not isinstance(value, dict):
        raise ValueError("顶层必须是对象")
    required = {"title", "priority", "steps"}
    if set(value) != required:
        raise ValueError("字段集合不符合契约")
    if not isinstance(value["title"], str) or not value["title"].strip():
        raise ValueError("title 必须是非空字符串")
    if value["priority"] not in {"low", "normal", "high"}:
        raise ValueError("priority 不合法")
    if not isinstance(value["steps"], list) or not all(
        isinstance(x, str) and x.strip() for x in value["steps"]
    ):
        raise ValueError("steps 必须是字符串数组")
    return value

if __name__ == "__main__":
    sample = '{"title":"检查备份","priority":"high","steps":["列出文件"]}'
    print(parse_task(sample))
~~~

## 运行与验证

保存为 llm_contract.py 并执行 python llm_contract.py；把 priority 改成 urgent，程序应拒绝该对象。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

重试只适用于可恢复故障并必须设上限；结构合法也不意味着内容真实或动作已获授权。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
