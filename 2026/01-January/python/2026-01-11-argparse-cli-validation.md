---
title: "用 argparse 构建有明确错误提示的命令行工具"
date: 2026-01-11
category: python
tags: [python, argparse, cli]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 argparse 构建有明确错误提示的命令行工具

命令行程序的输入也是外部输入。若把参数解析、业务逻辑和文件读写混在一起，错误提示容易不清楚，测试也需要模拟整段终端交互。标准库 argparse 已经提供类型转换、必需参数和帮助文本，足以覆盖不少小工具。

## 一个可运行的入口

```python
import argparse


def positive_int(value: str) -> int:
    try:
        number = int(value)
    except ValueError as exc:
        raise argparse.ArgumentTypeError("must be an integer") from exc
    if number <= 0:
        raise argparse.ArgumentTypeError("must be greater than zero")
    return number


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="Show a bounded sample count.")
    parser.add_argument("--count", type=positive_int, required=True)
    return parser


if __name__ == "__main__":
    args = build_parser().parse_args()
    print(f"count={args.count}")
```

保存为 `cli.py`，运行 `python cli.py --count 3` 应输出 `count=3`；运行 `python cli.py --count 0` 应显示参数错误并以非零状态退出。这里的预期行为来自 argparse 的参数校验机制；发布前仍应在目标 Python 版本中实际执行这两个场景。

## 设计原则

- 用独立函数校验复杂参数，方便单元测试。
- 错误信息告诉用户怎样修正，而不是只显示堆栈。
- 对路径、数字范围和枚举值做显式校验。
- 不要把敏感参数写进命令行，因为它们可能出现在进程列表或 shell 历史中。
- 对输出文件使用明确的覆盖策略，避免默认静默覆盖重要数据。

一个好的 CLI 不只是“能运行”，还要让错误输入有可预测的失败方式。
