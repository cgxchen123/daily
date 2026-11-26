---
title: "Agent 工作流别只记最终答案：用 trace_id 串起工具调用"
date: 2026-10-25
category: trends
tags: [trends, ai-agents, observability, tracing, python]
difficulty: intermediate
reading_time: 8
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

> 补档说明：本文归档日期为 2026-10-25，实际提交时间以 Git 提交记录为准，不表示文章在归档日期当天发布。

## 真实痛点

一个 Agent 任务可能依次执行检索、调用工具、整理结果。用户最后只看到“任务失败”，开发者却不知道是检索为空、工具超时，还是结果整理阶段抛错。若每个函数各写一份普通日志，又没有共同标识，排查时很难把同一次任务的记录拼起来。

解决办法不是把完整提示词、工具参数和模型回答全部写进日志，而是给一次工作流分配 **trace_id**，给每个步骤分配 **span_id**，再用 **parent_span_id** 表示父子关系。这样既能定位步骤，也能控制记录内容。

## 核心原理

**trace_id** 标识一次端到端工作流；**span_id** 标识其中一个操作；**parent_span_id** 记录它由哪个操作触发。每条记录至少保存步骤名、开始时间、耗时、状态和异常类型。耗时使用单调时钟计算，避免系统时间调整影响时长；时间戳则使用 UTC，方便跨机器对齐。

OpenTelemetry 的上下文传播机制也使用 trace 和 span 上下文关联跨服务活动；生产环境可用其 Python API/SDK 创建 span，而不是长期维护自制追踪协议。本文先用标准库实现一个本地 JSONL 版本，重点演示关联关系和失败记录。

## 完整可运行代码

保存为 agent_trace.py。需要 Python 3.10+，只使用标准库。程序不调用模型或网络服务，工具调用用本地函数模拟。运行成功流程会写入四条记录；模拟工具超时则记录失败步骤，并跳过后续步骤。

运行：

- `python agent_trace.py --self-test`：执行内置测试。
- `python agent_trace.py --output traces.jsonl`：写入成功流程。
- `python agent_trace.py --fail --output failed.jsonl`：模拟超时，预期退出码为 2。

~~~python
from __future__ import annotations

import argparse
import json
import sys
import tempfile
import unittest
import uuid
from contextlib import contextmanager
from datetime import datetime, timezone
from pathlib import Path
from time import perf_counter
from typing import Iterator


class JsonlTracer:
    """将一个工作流的步骤写成可关联的 JSONL 记录。"""

    def __init__(self, output: Path) -> None:
        self.output = output

    def _write(self, record: dict[str, object]) -> None:
        self.output.parent.mkdir(parents=True, exist_ok=True)
        line = json.dumps(record, ensure_ascii=False, separators=(",", ":"))
        with self.output.open("a", encoding="utf-8", newline="\n") as stream:
            stream.write(line + "\n")
            stream.flush()

    @contextmanager
    def span(
        self,
        trace_id: str,
        name: str,
        parent_span_id: str | None = None,
    ) -> Iterator[str]:
        span_id = uuid.uuid4().hex[:16]
        started_at = datetime.now(timezone.utc).isoformat()
        started = perf_counter()
        status = "ok"
        error_type: str | None = None

        try:
            yield span_id
        except Exception as exc:
            status = "error"
            error_type = type(exc).__name__
            raise
        finally:
            self._write(
                {
                    "trace_id": trace_id,
                    "span_id": span_id,
                    "parent_span_id": parent_span_id,
                    "name": name,
                    "status": status,
                    "error_type": error_type,
                    "started_at": started_at,
                    "duration_ms": round((perf_counter() - started) * 1000, 3),
                }
            )


def run_workflow(
    tracer: JsonlTracer,
    fail_tool: bool = False,
) -> tuple[str, str]:
    trace_id = uuid.uuid4().hex

    with tracer.span(trace_id, "workflow") as root_span_id:
        with tracer.span(trace_id, "retrieve", root_span_id):
            documents = ["doc-17", "doc-23"]

        with tracer.span(trace_id, "tool_call", root_span_id):
            if fail_tool:
                raise TimeoutError("simulated tool timeout")
            tool_result = "lookup-ok"

        with tracer.span(trace_id, "compose", root_span_id):
            answer = f"{tool_result}; retrieved={len(documents)}"

    return trace_id, answer


class TracerTests(unittest.TestCase):
    def test_success_records_parent_relationships(self) -> None:
        with tempfile.TemporaryDirectory() as directory:
            output = Path(directory) / "trace.jsonl"
            trace_id, answer = run_workflow(JsonlTracer(output))
            rows = [json.loads(line) for line in output.read_text(encoding="utf-8").splitlines()]

            self.assertEqual(answer, "lookup-ok; retrieved=2")
            self.assertEqual(len(rows), 4)
            self.assertTrue(all(row["trace_id"] == trace_id for row in rows))
            root = next(row for row in rows if row["name"] == "workflow")
            children = [row for row in rows if row["name"] != "workflow"]
            self.assertTrue(all(row["parent_span_id"] == root["span_id"] for row in children))
            self.assertTrue(all(row["status"] == "ok" for row in rows))

    def test_failure_is_recorded_and_later_step_is_skipped(self) -> None:
        with tempfile.TemporaryDirectory() as directory:
            output = Path(directory) / "trace.jsonl"
            with self.assertRaises(TimeoutError):
                run_workflow(JsonlTracer(output), fail_tool=True)

            rows = [json.loads(line) for line in output.read_text(encoding="utf-8").splitlines()]
            by_name = {row["name"]: row for row in rows}
            self.assertEqual(by_name["tool_call"]["status"], "error")
            self.assertEqual(by_name["tool_call"]["error_type"], "TimeoutError")
            self.assertEqual(by_name["workflow"]["status"], "error")
            self.assertNotIn("compose", by_name)


def run_tests() -> int:
    suite = unittest.defaultTestLoader.loadTestsFromTestCase(TracerTests)
    result = unittest.TextTestRunner(verbosity=2).run(suite)
    return 0 if result.wasSuccessful() else 1


def main() -> int:
    parser = argparse.ArgumentParser(description="记录 Agent 工作流的本地 trace")
    parser.add_argument("--output", type=Path, default=Path("agent-traces.jsonl"))
    parser.add_argument("--fail", action="store_true", help="模拟工具超时")
    parser.add_argument("--self-test", action="store_true", help="运行内置测试")
    args = parser.parse_args()

    if args.self_test:
        return run_tests()

    try:
        trace_id, answer = run_workflow(JsonlTracer(args.output), fail_tool=args.fail)
    except TimeoutError:
        print("工作流失败；错误步骤已写入 trace 文件", file=sys.stderr)
        return 2

    print(json.dumps({"status": "ok", "trace_id": trace_id, "answer": answer}, ensure_ascii=False))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
~~~

## 工程思考

追踪记录应该回答“哪一步、何时、耗时多久、成功还是失败”，而不是复制业务数据。不要把访问令牌、用户隐私、完整提示词或工具返回原文写入 JSONL；必要时只记录脱敏后的摘要或字段数量。异常只保存类型，详细堆栈应进入受控日志系统。

示例按单进程顺序追加文件，适合本地验证，不包含多进程写锁、日志轮转、采样和远端导出。若多个进程同时写同一个文件，不能假设每条记录天然不会交错。

不要把每个 token 或每次循环都建成一个 span。优先按“检索”“工具调用”“结果整理”等有意义阶段划分，名称保持稳定；不要把用户 ID、文档 ID 拼进 span 名称，否则聚合维度会膨胀。若一个步骤并行触发多个工具，子 span 仍共享同一 trace_id，但各自使用不同 span_id，父节点按真实调用关系填写。

排查时先按 trace_id 过滤，再依据 parent_span_id 还原层级。单条记录中的 duration_ms 只能说明这一次执行耗时，不能代替长期延迟分位数；样本不足时，不要从几条日志推断系统性能趋势。

## 升级方向

接入 OpenTelemetry 时，为工作流和每次工具调用创建嵌套 span，并使用官方 instrumentation 或 propagator 在服务边界传递上下文。再增加采样、脱敏、保留期限和告警规则。trace_id 只用于关联观测数据，不是授权凭证，也不能证明工具动作已获批准。

参考：

- [OpenTelemetry：Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [OpenTelemetry Python：Instrumentation](https://opentelemetry.io/docs/languages/python/instrumentation/)
