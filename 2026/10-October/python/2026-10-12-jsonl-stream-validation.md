---
title: "处理大文件别一次性读入内存：用 Python 流式校验 JSONL"
date: 2026-10-12
category: python
tags: [python, jsonl, streaming]
difficulty: beginner
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 处理大文件别一次性读入内存：用 Python 流式校验 JSONL

## 真实痛点

日志与模型输出常以 JSONL 保存。一次性读取整个文件会增加内存压力，遇到坏行也不容易定位。

## 核心原理

逐行读取、逐行解析，并把语法错误与业务结构错误分开。有效 JSON 不一定是对象，也不一定包含所需字段。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import json
from pathlib import Path

def validate(path: str) -> tuple[int, list[str]]:
    valid = 0
    errors = []
    with Path(path).open(encoding="utf-8") as stream:
        for number, line in enumerate(stream, 1):
            if not line.strip():
                continue
            try:
                value = json.loads(line)
            except json.JSONDecodeError as exc:
                errors.append(f"第 {number} 行 JSON 无效: {exc.msg}")
                continue
            if not isinstance(value, dict):
                errors.append(f"第 {number} 行必须是对象")
            elif "id" not in value:
                errors.append(f"第 {number} 行缺少 id")
            else:
                valid += 1
    return valid, errors

if __name__ == "__main__":
    count, errors = validate("sample.jsonl")
    print({"valid": count, "errors": errors})
~~~

## 运行与验证

创建 sample.jsonl，写入一行 {"id": 1}、一行 not-json，再运行脚本。程序应统计有效行并报告坏行号。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

错误列表本身可能随错误数量增长；超大文件应把错误逐条写入日志，而不是全部留在内存。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
