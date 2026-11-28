---
title: "AI 工具调用先做离线回放：用 JSONL 回归适配器"
date: 2026-10-27
category: ai-tools
tags: [ai-tools, tool-calling, replay, jsonl, unittest]
difficulty: intermediate
reading_time: 7
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

## 真实痛点

把模型接入工具调用后，问题往往出现在模型与业务代码之间：模型返回的工具名变了、参数类型不对，或者适配器把字段映射错了。直接连真实模型和外部服务做每次回归，不仅慢、成本不可控，还可能真的发送消息、写入数据或触发其他副作用。

排查时还容易混淆两类问题：是模型这次提出了不同动作，还是本地工具适配器处理错了？如果没有固定输入，修复后很难证明同一组请求是否稳定。只靠手工对话也容易漏掉边界：空字符串、额外字段、布尔值冒充整数、工具名拼错等情况，往往直到线上才出现。把这些输入固定下来，才能让每次改代码后的结果可比较。

## 核心原理

把已经脱敏的工具调用请求保存成 JSONL 测试样例，每行代表一次调用。离线回放器只允许调用本地注册表中的工具，并在执行前检查请求字段、工具名和参数类型。这样可以固定输入，重复验证解析、校验、分发和结果序列化。

这不是模型效果评测，也不能证明模型一定会生成正确请求；它只验证工具适配层。JSONL 的好处是每行独立、容易做版本控制，也能在失败时直接定位样例行号。示例工具只做纯计算和文本转换，不访问网络、不读写业务数据。真实工具应先用模拟实现替代，避免回放时意外触发副作用。

## 完整可运行代码

保存为 `tool_replay.py`，使用 Python 3.10+，仅依赖标准库：

```python
import argparse
import json
import sys
import tempfile
import unittest
from pathlib import Path
from typing import Any, TextIO


def add_numbers(arguments: dict[str, Any]) -> int:
    if set(arguments) != {"a", "b"}:
        raise ValueError("add_numbers 只接受 a、b 两个参数")
    if type(arguments["a"]) is not int or type(arguments["b"]) is not int:
        raise ValueError("a、b 必须是整数，不能是布尔值")
    return arguments["a"] + arguments["b"]


def uppercase(arguments: dict[str, Any]) -> str:
    if set(arguments) != {"text"} or not isinstance(arguments["text"], str):
        raise ValueError("uppercase 只接受 text 字符串")
    if len(arguments["text"]) > 200:
        raise ValueError("text 长度不能超过 200")
    return arguments["text"].upper()


TOOLS = {
    "add_numbers": add_numbers,
    "uppercase": uppercase,
}


def dispatch(payload: Any) -> dict[str, Any]:
    if not isinstance(payload, dict):
        raise ValueError("请求顶层必须是 JSON 对象")
    if set(payload) != {"id", "tool", "arguments"}:
        raise ValueError("请求字段必须且只能是 id、tool、arguments")

    request_id = payload["id"]
    tool_name = payload["tool"]
    arguments = payload["arguments"]

    if not isinstance(request_id, str) or not request_id.strip():
        raise ValueError("id 必须是非空字符串")
    if not isinstance(tool_name, str) or tool_name not in TOOLS:
        raise ValueError("未知工具")
    if not isinstance(arguments, dict):
        raise ValueError("arguments 必须是 JSON 对象")

    result = TOOLS[tool_name](arguments)
    return {"id": request_id, "ok": True, "result": result}


def process_line(raw: str, line_number: int) -> dict[str, Any]:
    try:
        payload = json.loads(raw)
    except json.JSONDecodeError:
        return {"line": line_number, "ok": False, "error": "invalid_json"}

    try:
        result = dispatch(payload)
    except ValueError as exc:
        return {"line": line_number, "ok": False, "error": str(exc)}

    return {"line": line_number, **result}


def run_file(path: Path, output: TextIO) -> int:
    failures = 0
    with path.open("r", encoding="utf-8") as stream:
        for line_number, raw in enumerate(stream, start=1):
            if not raw.strip():
                continue
            result = process_line(raw, line_number)
            print(json.dumps(result, ensure_ascii=False), file=output)
            failures += not result["ok"]
    return 1 if failures else 0


class ReplayTests(unittest.TestCase):
    def test_add_numbers(self) -> None:
        result = dispatch({
            "id": "t1",
            "tool": "add_numbers",
            "arguments": {"a": 2, "b": 3},
        })
        self.assertEqual(result["result"], 5)

    def test_boolean_is_not_integer(self) -> None:
        result = process_line(
            '{"id":"t2","tool":"add_numbers","arguments":{"a":true,"b":1}}',
            1,
        )
        self.assertFalse(result["ok"])

    def test_unknown_tool(self) -> None:
        result = process_line(
            '{"id":"t3","tool":"delete_all","arguments":{}}',
            1,
        )
        self.assertEqual(result["error"], "未知工具")

    def test_invalid_json(self) -> None:
        self.assertEqual(process_line("{", 4)["error"], "invalid_json")

    def test_uppercase_length_limit(self) -> None:
        result = process_line(
            json.dumps({
                "id": "t4",
                "tool": "uppercase",
                "arguments": {"text": "x" * 201},
            }),
            1,
        )
        self.assertFalse(result["ok"])

    def test_extra_argument_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            dispatch({
                "id": "t5",
                "tool": "add_numbers",
                "arguments": {"a": 1, "b": 2, "extra": 3},
            })

    def test_file_returns_failure_for_bad_line(self) -> None:
        with tempfile.TemporaryDirectory() as directory:
            path = Path(directory) / "calls.jsonl"
            path.write_text(
                '{"id":"t1","tool":"uppercase","arguments":{"text":"ok"}}\n'
                '{bad json}\n',
                encoding="utf-8",
            )
            from io import StringIO
            output = StringIO()
            self.assertEqual(run_file(path, output), 1)
            self.assertEqual(len(output.getvalue().splitlines()), 2)


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="离线回放 AI 工具调用样例")
    parser.add_argument("input", nargs="?", help="JSONL 样例文件")
    parser.add_argument("--self-test", action="store_true", help="运行内置回归测试")
    args = parser.parse_args(argv)

    if args.self_test:
        suite = unittest.defaultTestLoader.loadTestsFromTestCase(ReplayTests)
        result = unittest.TextTestRunner(verbosity=2).run(suite)
        return 0 if result.wasSuccessful() else 1

    if not args.input:
        parser.error("请提供 JSONL 文件，或使用 --self-test")

    try:
        return run_file(Path(args.input), sys.stdout)
    except OSError as exc:
        print(f"读取失败：{exc}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    raise SystemExit(main())
```

创建 `demo.jsonl`，每行一个请求：

```jsonl
{"id":"case-1","tool":"add_numbers","arguments":{"a":2,"b":3}}
{"id":"case-2","tool":"uppercase","arguments":{"text":"hello"}}
```

运行内置测试，再回放样例：

```bash
python tool_replay.py --self-test
python tool_replay.py demo.jsonl
```

## 工程思考

回放样例应来自经过脱敏的请求，不要保存密钥、完整提示词、个人信息或真实业务数据。工具注册表是本地白名单，不能把模型给出的字符串直接当函数名执行。参数校验也要检查精确类型：Python 中 `bool` 是 `int` 的子类，所以金额、数量等整数参数不能只用 `isinstance(value, int)`。这一行为见 [Python 3.10 官方文档](https://docs.python.org/3.10/library/functions.html#bool)。

如果回放失败，先看输出中的行号和错误类型，再判断是样例格式、参数契约还是工具实现的问题。不要为了让测试通过而自动忽略失败行。样例还应有明确的维护责任：删除过时字段前先确认调用契约已经变化，更新样例时保留能复现旧问题的最小输入。

## 升级方向

可以为样例增加 `schema_version` 和预期结果，比较每次适配器改动前后的输出；再把少量代表性样例接入自动测试。样例变更应和工具契约一起审查，避免为了让新代码通过而同步改掉预期结果。涉及真实模型 API、数据库或消息发送的测试应单独设置明确开关和隔离环境。离线回放的价值，是先把本地边界测稳定，而不是假装它覆盖了整条 Agent 链路。
