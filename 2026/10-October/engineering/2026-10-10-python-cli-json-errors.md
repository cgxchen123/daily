---
title: "别让命令行工具只会报错：用 Python 统一日志、JSON 输出与退出码"
date: 2026-10-10
category: engineering
tags: [python, cli, logging, json, error-handling, engineering]
difficulty: beginner
reading_time: 7 min
python_version: "3.10+"
dependencies: []
series: null
---

# 别让命令行工具只会报错：用 Python 统一日志、JSON 输出与退出码

## 真实痛点

脚本在本地运行时，打印一句“处理失败”似乎够用；但当它进入定时任务、CI 或被另一个程序调用时，问题就出现了：错误信息混在正常输出里，调用方无法稳定判断成功与否，日志缺少上下文，失败后也不知道应该重试还是修正输入。

命令行程序至少要给出三种明确的信号：标准输出承载结果，标准错误承载诊断信息，退出码供 Shell、调度器和 CI 判断执行结果。下面实现一个完整工具：读取 JSON 文件、验证必需字段、统计记录数，并支持人类可读或 JSON 输出。它只依赖 Python 标准库。

## 核心原理

1. 把可预期的输入错误与未预期的处理错误分开。
2. 用 `logging` 将诊断信息写到标准错误。
3. JSON 模式下，标准输出只输出合法 JSON，不混入日志。
4. 固定退出码：`0` 成功、`2` 参数或输入错误、`1` 未预期错误。
5. 默认不显示异常堆栈；排查时可使用 `--debug` 查看详细日志。

## 完整可运行代码

保存为 `json_report.py`。需要 Python 3.10 或更高版本，无第三方依赖。

```python
#!/usr/bin/env python3
"""Validate a JSON object and report its record count."""

from __future__ import annotations

import argparse
import json
import logging
import sys
from pathlib import Path
from typing import Any


def load_document(path: Path) -> dict[str, Any]:
    """Load a JSON object and validate its basic shape."""
    try:
        with path.open("r", encoding="utf-8") as handle:
            value = json.load(handle)
    except OSError as exc:
        raise ValueError(f"cannot read input file {path}: {exc}") from exc
    except json.JSONDecodeError as exc:
        raise ValueError(
            f"invalid JSON at line {exc.lineno}, column {exc.colno}"
        ) from exc

    if not isinstance(value, dict):
        raise ValueError("top-level JSON value must be an object")

    records = value.get("records")
    if not isinstance(records, list):
        raise ValueError("required field 'records' must be a JSON array")
    return value


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description="Validate a JSON document and count its records."
    )
    parser.add_argument("input", type=Path, help="path to a JSON input file")
    parser.add_argument(
        "--format", choices=("text", "json"), default="text",
        help="output format (default: text)"
    )
    parser.add_argument(
        "--debug", action="store_true",
        help="include a traceback for unexpected errors"
    )
    return parser


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    logging.basicConfig(
        level=logging.DEBUG if args.debug else logging.WARNING,
        format="%(levelname)s: %(message)s",
        stream=sys.stderr,
    )

    try:
        document = load_document(args.input)
        result = {
            "ok": True,
            "record_count": len(document["records"]),
            "keys": sorted(document.keys()),
        }
    except ValueError as exc:
        logging.error("%s", exc)
        return 2
    except Exception:
        logging.exception("unexpected processing error")
        return 1

    if args.format == "json":
        print(json.dumps(result, ensure_ascii=False, sort_keys=True))
    else:
        print(f"Valid JSON object; records={result['record_count']}")
        print(f"Top-level keys: {', '.join(result['keys'])}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## 运行与验证

创建 `sample.json`：

```json
{
  "source": "demo",
  "records": [{"id": 1}, {"id": 2}]
}
```

运行：

```bash
python json_report.py sample.json
python json_report.py sample.json --format json
```

JSON 模式的标准输出应为：

```json
{"keys": ["records", "source"], "ok": true, "record_count": 2}
```

再测试错误路径：

```bash
python json_report.py missing.json
echo $?
```

文件不存在时，诊断信息写入标准错误，退出码为 `2`。Windows PowerShell 可用 `$LASTEXITCODE` 查看退出码。要让下游程序稳定解析 JSON，应把标准输出和标准错误分开重定向。不要把 `2>&1` 合并后的内容直接当成 JSON，因为诊断文本不是 JSON。

## 工程思考

错误处理是程序对外接口的一部分，而不只是几行提示文字。退出码需要稳定，标准输出要遵守格式契约，日志则应包含足够定位问题的上下文。示例将可预期的输入问题映射为退出码 2，把未预期异常映射为退出码 1；没有捕获异常后继续假装成功。

真实项目还应按业务定义错误类型，避免把所有异常都笼统归类。日志也可能包含文件路径或其他敏感信息，进入共享 CI 日志之前，应检查输出，避免打印令牌、个人数据或完整敏感配置。

## 升级方向

- 为输入校验和退出码增加自动化测试。
- 大文件场景加入流式处理，避免一次性把全部内容读入内存。
- 如果工具由其他程序调用，固定 JSON Schema，并对标准输出做契约测试。
- 在调度器中记录退出码、执行时长和输入摘要，便于定位重复失败；不要把“进程启动过”当成“任务成功”。
