---
title: "Python 长任务如何优雅停机：完成当前任务，再从结果文件续跑"
date: 2026-10-24
category: engineering
tags: [python, signal, graceful-shutdown, batch-processing, resumable-jobs]
difficulty: intermediate
reading_time: 14
python_version: "3.10+"
dependencies: []
series: "Daily-Tech-Share"
---

> 补档说明：本文归档日期为 2026-10-24，实际 Git 提交时间为 2026-11-24。本篇是历史日期补档，不表示文章在 10 月 24 日当天发布。

# Python 长任务如何优雅停机：完成当前任务，再从结果文件续跑

## 真实痛点

一个批处理脚本可能要处理几千个文件、导入一批数据，或者逐条调用外部服务。开发时按下 Ctrl+C，程序似乎只是停了；但真正麻烦的是：它可能刚读完输入、还没写结果，或者已经完成了副作用却没有记录完成状态。下一次启动时，操作者无法判断哪些任务已经完成，哪些任务应该重做。

常见的两个极端都不好：

- 直接忽略中断信号：部署、升级或人工停止时，进程迟迟不退出。
- 收到信号立即抛异常：当前任务可能只做了一半，输出文件留下不完整记录，重启后又重复执行。

更可控的目标不是“收到信号就立刻杀掉程序”，而是把停止过程拆成三个动作：**不再领取新任务、让当前任务走到一个明确的结束点、把已完成结果可靠地记录下来**。如果任务还没全部完成，下一次启动能够识别已完成项并继续。

本文用一个本地文件哈希批处理器演示这套边界。输入是 JSONL 任务清单，每行包含任务 ID 和相对文件路径；程序计算文件大小与 SHA-256，并把每项结果追加到 JSONL 文件。它不访问网络、不删除源文件，也不需要第三方依赖。

## 核心原理

### 1. 信号处理函数只表达“想停”，不做清理工作

Python 的信号处理函数总是在主解释器的主线程中执行；只有主线程可以安装处理函数。信号处理函数也可能因为解释器正在执行较长的 C 代码而延迟运行。因此，不应把它当成一个可以随时打断任意操作的事务回滚钩子。

本文的处理函数只记录第一个信号编号。它不打印日志、不写文件、不关闭资源，也不获取锁。主循环在每个任务开始前检查这个标记：如果已经收到 SIGINT 或 SIGTERM，就不再领取下一项。

### 2. “当前任务完成”必须有明确边界

程序在开始一项任务后，会继续完成文件读取、哈希计算和结果写入。信号如果在任务处理中到达，只改变停止标记，不会让当前任务直接跳到异常分支。当前项完成后，循环才停止。

这不代表任何任务都能无限期运行。若任务卡在不可中断的系统调用、外部服务或长时间 C 扩展中，优雅退出仍会等待它返回。真实项目还应为单项任务设置超时、取消协议或最大执行时间。

### 3. 先持久化完成记录，再把 ID 视为已完成

结果文件采用追加式 JSONL。每条成功记录写入后，程序调用 flush，再调用 os.fsync，让 Python 缓冲区的数据先交给操作系统，并请求操作系统同步文件描述符。重启时，程序只把 status 为 ok 的 ID 视为已完成；error 记录不会被当作成功，因此下次启动可以重试。

任务 ID 必须稳定，并且同一个 ID 不能悄悄换成另一条路径。否则，旧结果可能被错误地用于跳过新任务。代码同时检查输入目录边界：相对路径解析后必须仍在输入清单所在目录内，符号链接指向目录外也会被拒绝。

需要明确边界：fsync 不是数据库事务，也不能保证多个进程同时追加同一个结果文件时不互相干扰。这个示例按单进程运行设计；如果要多进程并发、跨机器调度或保证复杂业务副作用，应该升级到有事务和唯一约束的存储。

## 完整可运行代码

### 1. 批处理程序 graceful_batch.py

将下面代码保存为 graceful_batch.py。只使用 Python 3.10+ 标准库。

~~~python
from __future__ import annotations

import argparse
import hashlib
import json
import math
import os
import signal
import sys
import time
from pathlib import Path
from typing import Any


def non_negative_float(value: str) -> float:
    try:
        number = float(value)
    except ValueError as exc:
        raise argparse.ArgumentTypeError("必须是数字") from exc
    if not math.isfinite(number) or number < 0:
        raise argparse.ArgumentTypeError("必须是有限的非负数")
    return number


def load_completed(output_path: Path) -> dict[str, str]:
    completed: dict[str, str] = {}
    if not output_path.exists():
        return completed

    with output_path.open("r", encoding="utf-8") as stream:
        for line_number, line in enumerate(stream, start=1):
            if not line.strip():
                continue
            try:
                record = json.loads(line)
            except json.JSONDecodeError as exc:
                raise ValueError(
                    f"结果文件第 {line_number} 行不是合法 JSON"
                ) from exc

            if not isinstance(record, dict):
                raise ValueError(f"结果文件第 {line_number} 行必须是对象")

            task_id = record.get("id")
            status = record.get("status")
            relative_path = record.get("path")
            if not all(
                isinstance(value, str)
                for value in (task_id, status, relative_path)
            ):
                raise ValueError(
                    f"结果文件第 {line_number} 行缺少 id/status/path"
                )
            if status not in {"ok", "error"}:
                raise ValueError(
                    f"结果文件第 {line_number} 行的 status 不合法"
                )

            if status == "ok":
                previous = completed.get(task_id)
                if previous is not None and previous != relative_path:
                    raise ValueError(
                        f"结果文件中的任务 {task_id!r} 对应了不同路径"
                    )
                completed[task_id] = relative_path

    return completed


def parse_task(
    line: str,
    line_number: int,
    base_dir: Path,
) -> tuple[str, str, Path]:
    try:
        record = json.loads(line)
    except json.JSONDecodeError as exc:
        raise ValueError(
            f"输入文件第 {line_number} 行不是合法 JSON"
        ) from exc

    if not isinstance(record, dict) or set(record) != {"id", "path"}:
        raise ValueError(
            f"输入文件第 {line_number} 行必须且只能包含 id、path"
        )

    task_id = record["id"]
    raw_path = record["path"]
    if not isinstance(task_id, str) or not task_id.strip():
        raise ValueError(
            f"输入文件第 {line_number} 行的 id 必须是非空字符串"
        )
    if not isinstance(raw_path, str) or not raw_path.strip():
        raise ValueError(
            f"输入文件第 {line_number} 行的 path 必须是非空字符串"
        )

    relative = Path(raw_path)
    if relative.is_absolute():
        raise ValueError(
            f"输入文件第 {line_number} 行不允许绝对路径"
        )

    candidate = (base_dir / relative).resolve()
    try:
        normalized = candidate.relative_to(base_dir)
    except ValueError as exc:
        raise ValueError(
            f"输入文件第 {line_number} 行的路径超出输入目录"
        ) from exc

    return task_id.strip(), normalized.as_posix(), candidate


def hash_file(path: Path) -> tuple[int, str]:
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for chunk in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(chunk)
    return path.stat().st_size, digest.hexdigest()


def append_record(stream: Any, record: dict[str, Any]) -> None:
    stream.write(
        json.dumps(record, ensure_ascii=False, sort_keys=True) + "\n"
    )
    stream.flush()
    os.fsync(stream.fileno())


def run(input_path: Path, output_path: Path, delay: float = 0.0) -> int:
    input_path = input_path.resolve(strict=True)
    if not input_path.is_file():
        raise ValueError("输入路径必须是文件")

    output_path = output_path.resolve()
    if input_path == output_path:
        raise ValueError("输入文件和输出文件不能是同一个文件")

    base_dir = input_path.parent.resolve()
    output_path.parent.mkdir(parents=True, exist_ok=True)
    completed = load_completed(output_path)

    received_signal: int | None = None

    def request_stop(signum: int, frame: object) -> None:
        nonlocal received_signal
        if received_signal is None:
            received_signal = signum

    watched_signals = [signal.SIGINT]
    if hasattr(signal, "SIGTERM"):
        watched_signals.append(signal.SIGTERM)

    previous_handlers: dict[int, Any] = {}
    completed_count = 0
    skipped_count = 0
    error_count = 0
    seen_input: dict[str, str] = {}

    try:
        for signum in watched_signals:
            previous_handlers[signum] = signal.getsignal(signum)
            signal.signal(signum, request_stop)

        with input_path.open("r", encoding="utf-8") as source, output_path.open(
            "a", encoding="utf-8", newline="\n"
        ) as output:
            for line_number, line in enumerate(source, start=1):
                if not line.strip():
                    continue

                task_id, relative_path, file_path = parse_task(
                    line, line_number, base_dir
                )

                previous_input = seen_input.get(task_id)
                if previous_input is not None:
                    if previous_input != relative_path:
                        raise ValueError(
                            f"任务 id {task_id!r} 在输入中对应了不同路径"
                        )
                    skipped_count += 1
                    continue
                seen_input[task_id] = relative_path

                previous_output = completed.get(task_id)
                if previous_output is not None:
                    if previous_output != relative_path:
                        raise ValueError(
                            f"已完成任务 {task_id!r} 的路径发生变化"
                        )
                    skipped_count += 1
                    continue

                if received_signal is not None:
                    break

                # 信号若在这里到达，当前任务仍会完成；下一轮才停止接收新任务。
                if delay:
                    time.sleep(delay)

                try:
                    size_bytes, sha256 = hash_file(file_path)
                    result = {
                        "id": task_id,
                        "path": relative_path,
                        "status": "ok",
                        "size_bytes": size_bytes,
                        "sha256": sha256,
                    }
                    completed[task_id] = relative_path
                    completed_count += 1
                except OSError as exc:
                    result = {
                        "id": task_id,
                        "path": relative_path,
                        "status": "error",
                        "error_type": type(exc).__name__,
                    }
                    error_count += 1

                append_record(output, result)

    finally:
        for signum, previous_handler in previous_handlers.items():
            signal.signal(signum, previous_handler)

    print(
        f"完成={completed_count} 跳过={skipped_count} 失败={error_count}",
        file=sys.stderr,
    )

    if received_signal is not None:
        try:
            signal_name = signal.Signals(received_signal).name
        except ValueError:
            signal_name = str(received_signal)
        print(
            f"收到 {signal_name}，当前任务已收尾，停止接收新任务",
            file=sys.stderr,
        )
        return 128 + received_signal

    return 1 if error_count else 0


def main() -> int:
    parser = argparse.ArgumentParser(
        description="逐项计算文件 SHA-256，支持优雅停止和断点续跑"
    )
    parser.add_argument("input", type=Path, help="JSONL 任务清单")
    parser.add_argument("output", type=Path, help="JSONL 结果文件")
    parser.add_argument(
        "--delay",
        type=non_negative_float,
        default=0.0,
        help="每项开始后的等待秒数，仅用于演示和测试",
    )
    args = parser.parse_args()

    try:
        return run(args.input, args.output, args.delay)
    except (OSError, ValueError) as exc:
        print(f"错误：{exc}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    raise SystemExit(main())
~~~

### 2. 回归测试 test_graceful_batch.py

将下面测试文件与 graceful_batch.py 放在同一目录。测试覆盖正常完成、重复启动、失败重试、路径越界、损坏结果文件和 SIGINT 中断后续跑。信号子进程测试仅在 POSIX 环境执行，其他平台会明确跳过这一项。

~~~python
import json
import os
import signal
import subprocess
import sys
import tempfile
import time
import unittest
from pathlib import Path


SCRIPT = Path(__file__).with_name("graceful_batch.py")


def write_tasks(path: Path, tasks: list[dict[str, str]]) -> None:
    path.write_text(
        "".join(json.dumps(task) + "\n" for task in tasks),
        encoding="utf-8",
    )


def read_results(path: Path) -> list[dict[str, object]]:
    if not path.exists():
        return []
    records = []
    for line in path.read_text(encoding="utf-8").splitlines():
        if not line:
            continue
        try:
            records.append(json.loads(line))
        except json.JSONDecodeError:
            continue
    return records


class GracefulBatchTests(unittest.TestCase):
    def run_cli(
        self,
        input_path: Path,
        output_path: Path,
        *extra: str,
    ) -> subprocess.CompletedProcess[str]:
        return subprocess.run(
            [
                sys.executable,
                str(SCRIPT),
                str(input_path),
                str(output_path),
                *extra,
            ],
            capture_output=True,
            text=True,
            check=False,
        )

    def test_success_and_resume_without_duplicate_output(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            (root / "a.txt").write_text("alpha", encoding="utf-8")
            (root / "b.txt").write_text("beta", encoding="utf-8")
            source = root / "tasks.jsonl"
            output = root / "results.jsonl"
            write_tasks(
                source,
                [
                    {"id": "a", "path": "a.txt"},
                    {"id": "b", "path": "b.txt"},
                ],
            )

            first = self.run_cli(source, output)
            self.assertEqual(first.returncode, 0, first.stderr)
            self.assertEqual(len(read_results(output)), 2)

            second = self.run_cli(source, output)
            self.assertEqual(second.returncode, 0, second.stderr)
            self.assertEqual(len(read_results(output)), 2)
            self.assertIn("跳过=2", second.stderr)

    def test_failed_item_is_retried_after_file_appears(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            source = root / "tasks.jsonl"
            output = root / "results.jsonl"
            write_tasks(
                source,
                [{"id": "late", "path": "later.txt"}],
            )

            first = self.run_cli(source, output)
            self.assertEqual(first.returncode, 1, first.stderr)
            self.assertEqual(read_results(output)[0]["status"], "error")

            (root / "later.txt").write_text("now exists", encoding="utf-8")
            second = self.run_cli(source, output)
            self.assertEqual(second.returncode, 0, second.stderr)
            self.assertEqual(
                [row["status"] for row in read_results(output)],
                ["error", "ok"],
            )

    def test_path_traversal_is_rejected(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            source = root / "tasks.jsonl"
            output = root / "results.jsonl"
            write_tasks(
                source,
                [{"id": "outside", "path": "../outside.txt"}],
            )

            result = self.run_cli(source, output)
            self.assertEqual(result.returncode, 2)
            self.assertIn("超出输入目录", result.stderr)
            self.assertEqual(read_results(output), [])

    def test_corrupted_existing_output_fails_closed(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            source = root / "tasks.jsonl"
            output = root / "results.jsonl"
            (root / "a.txt").write_text("alpha", encoding="utf-8")
            write_tasks(
                source,
                [{"id": "a", "path": "a.txt"}],
            )
            output.write_text('{"id": "broken"\n', encoding="utf-8")

            result = self.run_cli(source, output)
            self.assertEqual(result.returncode, 2)
            self.assertIn("不是合法 JSON", result.stderr)

    @unittest.skipUnless(
        os.name == "posix",
        "该信号子进程测试在 POSIX 环境运行",
    )
    def test_sigint_finishes_current_item_then_resume(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            tasks = []
            for index in range(5):
                name = f"file-{index}.txt"
                (root / name).write_text(
                    f"payload-{index}",
                    encoding="utf-8",
                )
                tasks.append(
                    {"id": f"task-{index}", "path": name}
                )

            source = root / "tasks.jsonl"
            output = root / "results.jsonl"
            write_tasks(source, tasks)

            process = subprocess.Popen(
                [
                    sys.executable,
                    str(SCRIPT),
                    str(source),
                    str(output),
                    "--delay",
                    "0.6",
                ],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
            )

            deadline = time.monotonic() + 8
            while time.monotonic() < deadline:
                if len(read_results(output)) >= 1:
                    break
                if process.poll() is not None:
                    self.fail("子进程在首条结果写入前退出")
                time.sleep(0.02)
            else:
                process.kill()
                self.fail("等待首条结果超时")

            time.sleep(0.1)
            process.send_signal(signal.SIGINT)
            _, stderr = process.communicate(timeout=8)
            self.assertEqual(
                process.returncode,
                128 + signal.SIGINT,
                stderr,
            )

            partial = read_results(output)
            self.assertEqual(len(partial), 2)
            self.assertEqual(
                [row["id"] for row in partial],
                ["task-0", "task-1"],
            )

            resumed = self.run_cli(source, output)
            self.assertEqual(resumed.returncode, 0, resumed.stderr)
            final_records = read_results(output)
            self.assertEqual(len(final_records), 5)
            self.assertEqual(
                len({row["id"] for row in final_records}),
                5,
            )


if __name__ == "__main__":
    unittest.main(verbosity=2)
~~~

## 运行与验证

目录示例：

~~~text
project/
├── graceful_batch.py
├── test_graceful_batch.py
├── tasks.jsonl
└── files/
    ├── a.txt
    └── b.txt
~~~

任务清单 tasks.jsonl：

~~~json
{"id": "file-a", "path": "files/a.txt"}
{"id": "file-b", "path": "files/b.txt"}
~~~

在 project 目录执行：

~~~bash
python graceful_batch.py tasks.jsonl results.jsonl
~~~

正常完成时退出码为 0。结果文件每行对应一项，成功记录包含 id、path、status、size_bytes 和 sha256。若某个文件暂时不存在，程序写入 status 为 error 的记录，继续处理其他项，最后以退出码 1 结束；修复文件后重新运行，该 ID 会再次尝试。

在长任务运行期间按 Ctrl+C，程序记录 SIGINT，完成已经开始的那一项后停止领取下一项，并以 130 退出。收到 SIGTERM 时，退出码按 128 加信号编号计算。再次运行时，已成功记录的任务会被跳过，未成功的任务继续执行。

运行回归测试：

~~~bash
python -m unittest -v test_graceful_batch
~~~

本次在 Python 3.13.5 环境中实际执行了这 5 个测试：正常完成与重复启动、失败项修复后重试、路径穿越拒绝、损坏结果文件拒绝、POSIX 下 SIGINT 完成当前项并恢复剩余任务。测试进程退出码为 0，5 项测试全部通过。Python 3.10+ 是代码声明的最低版本；本次实际运行版本为 3.13.5，不把本次结果外推成所有操作系统和 Python 小版本均已验证。

## 工程思考

**第一，任务 ID 是恢复协议的一部分。** 如果每次启动都重新生成随机 ID，程序就无法识别之前的成功项。ID 应来自稳定业务主键，且同一个 ID 对应的输入内容不能悄悄变化。示例还记录了相对路径，发现已完成 ID 被映射到另一条路径时会停止，而不是直接跳过。

**第二，成功记录必须发生在任务真正完成之后。** 如果真实业务是“调用支付接口、发送邮件、写数据库”，仅把结果写入本地文件并不能保证外部副作用只发生一次。进程可能在外部服务成功后、写本地结果前崩溃。此时需要外部服务支持幂等键，或者使用事务性消息、数据库唯一约束和可查询的操作状态。不要把本地 JSONL 检查点误称为端到端 exactly-once。

**第三，fsync 有成本，也有边界。** 每条记录都 flush 和 fsync，恢复点更细，但吞吐会下降。对于可重算、低风险任务，可以按固定条数或时间批量同步；对于不可重复的副作用，应优先设计业务幂等和事务，而不是只调大 fsync 间隔。突然断电、文件系统和存储设备的写入语义也需要在目标环境验证。

**第四，单进程示例不等于并发安全。** 两个进程同时读取同一个结果文件、各自判断某个 ID 未完成，然后同时执行，会产生竞争。若要并行运行，应引入文件锁或数据库唯一键，并让“领取任务”和“记录完成”在同一套并发控制下完成。

## 升级方向与适用边界

- 任务量很大时，可将完成状态放入 SQLite，以任务 ID 建唯一索引，并在事务中写入结果。
- 任务耗时不可预测时，为每项任务增加 deadline、超时和可取消接口；停止等待也应有上限。
- 需要多进程或多机器时，增加任务租约、锁过期和心跳，防止进程异常退出后任务永久占用。
- 需要恢复外部副作用时，为每个业务动作生成稳定幂等键，并保存请求状态、外部响应标识和补偿策略。
- 对结果文件进行轮转、备份和定期校验；若出现不完整的最后一行，示例会拒绝继续读取，应该先人工检查或从可信备份恢复，不能直接忽略损坏数据。

这个示例解决的是单进程、本地文件任务的可控停止与恢复。SIGKILL、机器断电、磁盘故障、外部系统已经成功但本地尚未记录等情况，仍需要更强的持久化和业务协议。

## 参考资料

- [Python 3.10 signal 模块文档](https://docs.python.org/3.10/library/signal.html)：信号处理函数执行线程、安装处理函数的线程限制及平台差异。
- [Python 3.10 os.fsync 文档](https://docs.python.org/3.10/library/os.html#os.fsync)：文件缓冲区 flush 与文件描述符同步的调用顺序。
- [Python 3.10 pathlib.Path.resolve 文档](https://docs.python.org/3.10/library/pathlib.html#pathlib.Path.resolve)：路径解析和符号链接处理。
