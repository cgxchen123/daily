---
title: "批量重命名前先做计划：用 Python 预览冲突并回滚"
date: 2026-10-26
category: python
tags: [python, pathlib, filesystem, dry-run, rollback]
difficulty: intermediate
reading_time: 9
python_version: "3.13+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

> 补档说明：本文归档日期为 2026-10-26，实际发布以 Git 提交时间为准，不表示文章在归档日期当天发布。

## 真实痛点

整理下载目录、项目素材或导出文件时，经常需要给一批文件统一加前缀、改后缀或按规则改名。最容易出问题的做法，是写一个循环，边遍历边调用 `rename()`：前几个文件已经改好，后面才发现目标文件名早就存在；更糟的是，不同操作系统对目标已存在时的处理并不完全一样，脚本可能覆盖文件，也可能中途报错。

批量重命名不是“把字符串拼出来”这么简单。它同时涉及文件筛选、目标路径规划、冲突检查、执行顺序和失败恢复。尤其是下载目录里常有同名文件、临时文件和快捷链接，不能只凭文件名看起来合理就直接动手。

更稳妥的办法是把操作拆成两个阶段：先生成一份确定的计划，只展示“源文件 → 目标文件”；确认计划没有冲突后，再由用户显式传入 `--apply` 执行。默认运行只预览，不修改任何文件。这样至少能让操作者在发生副作用前看见完整变更范围。

## 核心原理

第一步是限定范围。示例只扫描指定目录的第一层，只处理普通文件，不递归进入子目录，也跳过符号链接和已经带有目标前缀的文件。递归扫描看起来方便，但它会扩大影响范围；在没有明确需求时，越小的扫描范围越容易审查。文件按名称排序，确保同一目录在没有变化时生成稳定计划，便于比较和复核。

第二步是预检目标。程序先为每个源文件计算目标路径，再检查目标是否已存在，并检查计划里是否出现重复目标。只要发现一个冲突，就整体停止，不开始处理第一个文件。这样可以避免“前面成功、后面才发现撞名”的常见半成品状态。

第三步是避免覆盖。不能简单认为 `Path.rename()` 在所有平台上都会拒绝已存在的目标：Python 文档说明，Unix 上目标文件可能被静默替换，而 Windows 上会抛出 `FileExistsError`。为了让示例不依赖这种差异，程序使用 `os.link(source, destination)` 先排他地创建目标硬链接，再删除源目录项；如果目标已存在，创建链接会失败，不会主动覆盖目标。硬链接要求源和目标位于同一文件系统，且文件系统支持硬链接；本例把目标放在源文件同一目录，并在不支持时安全报错。

第四步是处理执行中断。批量操作不是文件系统事务，进程被强制终止或机器掉电时，无法承诺整个目录一次性回滚。示例能处理普通运行时异常：记录已经完成的项目，发生错误时按相反顺序尝试恢复。若恢复本身失败，程序会明确报告回滚不完整，而不是假装所有文件都没变化。执行前仍应备份重要目录，并确保没有其他程序同时改动该目录。

最后要区分“预览通过”和“执行成功”。预览只证明当前时刻的计划没有发现冲突；它不能锁住目录，也不能保证稍后执行时目录仍然不变。真正执行时，程序会再次检查源和目标，并依赖目标创建操作拒绝已有路径。对于多人共享目录或持续写入的目录，还需要外部锁或维护窗口。

## 完整可运行代码

保存为 `rename_tool.py`。要求 Python 3.13+，只使用标准库，无需安装第三方依赖。默认只预览 `.txt` 文件并添加 `archived_` 前缀；执行时必须显式加 `--apply`。

先运行测试：

```bash
python rename_tool.py --self-test
```

预览当前目录：

```bash
python rename_tool.py ./incoming
```

预览其他扩展名：

```bash
python rename_tool.py ./incoming --suffix .jpg --prefix reviewed_
```

确认输出清单无误后才执行：

```bash
python rename_tool.py ./incoming --apply
```

本地验证环境为 Python 3.13.5。代码同时包含四类测试：预览不修改文件、目标冲突时整体停止、正常执行后内容保持不变，以及模拟第二个文件失败时回滚第一个文件。测试只在临时目录中创建文件，不会触碰真实业务目录。

~~~python
from __future__ import annotations

import argparse
import os
import sys
import tempfile
import unittest
from pathlib import Path
from unittest.mock import patch


def build_plan(directory: Path, suffix: str, prefix: str) -> list[tuple[Path, Path]]:
    if not directory.is_dir() or directory.is_symlink():
        raise NotADirectoryError(directory)
    if not suffix.startswith(".") or len(suffix) < 2:
        raise ValueError("suffix 必须是以点开头的扩展名")
    if not prefix.strip() or prefix in {".", ".."} or "/" in prefix or "\\" in prefix:
        raise ValueError("prefix 必须是非空且不含路径分隔符的文件名前缀")

    sources = sorted(
        path for path in directory.iterdir()
        if path.is_file() and not path.is_symlink()
        and path.suffix.casefold() == suffix.casefold()
        and not path.name.startswith(prefix)
    )
    plan = [(source, source.with_name(prefix + source.name)) for source in sources]

    destinations = [destination for _, destination in plan]
    if len(set(destinations)) != len(destinations):
        raise FileExistsError("计划中存在重复目标路径")
    occupied = [destination for destination in destinations if destination.exists()]
    if occupied:
        names = ", ".join(path.name for path in occupied)
        raise FileExistsError(f"目标文件已存在：{names}")
    return plan


def link_then_unlink(source: Path, destination: Path) -> None:
    """用排他创建目标链接的方式避免覆盖已有目标文件。"""
    os.link(source, destination)
    try:
        source.unlink()
    except OSError:
        try:
            destination.unlink()
        except OSError as cleanup_error:
            raise RuntimeError(
                f"源文件删除失败，且无法清理新链接：{destination}"
            ) from cleanup_error
        raise


def apply_plan(plan: list[tuple[Path, Path]]) -> list[tuple[Path, Path]]:
    completed: list[tuple[Path, Path]] = []
    try:
        for source, destination in plan:
            if not source.is_file() or source.is_symlink():
                raise FileNotFoundError(f"源文件已变化：{source}")
            if destination.exists():
                raise FileExistsError(f"目标文件已出现：{destination}")
            link_then_unlink(source, destination)
            completed.append((source, destination))
    except (OSError, RuntimeError) as original_error:
        rollback_errors: list[str] = []
        for source, destination in reversed(completed):
            try:
                link_then_unlink(destination, source)
            except (OSError, RuntimeError) as rollback_error:
                rollback_errors.append(f"{destination.name}: {rollback_error}")
        if rollback_errors:
            raise RuntimeError(
                "批量操作失败，且回滚未全部完成：" + "; ".join(rollback_errors)
            ) from original_error
        raise
    return completed


class RenameTests(unittest.TestCase):
    def test_plan_is_preview_only_and_sorted(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            (root / "b.TXT").write_text("b", encoding="utf-8")
            (root / "a.txt").write_text("a", encoding="utf-8")
            (root / "archived_old.txt").write_text("old", encoding="utf-8")
            plan = build_plan(root, ".txt", "archived_")
            self.assertEqual(
                [source.name for source, _ in plan], ["a.txt", "b.TXT"]
            )
            self.assertTrue((root / "a.txt").exists())
            self.assertFalse((root / "archived_a.txt").exists())

    def test_existing_destination_blocks_everything(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            source = root / "report.txt"
            source.write_text("source", encoding="utf-8")
            (root / "archived_report.txt").write_text("keep", encoding="utf-8")
            with self.assertRaises(FileExistsError):
                build_plan(root, ".txt", "archived_")
            self.assertEqual(source.read_text(encoding="utf-8"), "source")

    def test_apply_creates_new_names_without_overwrite(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            source = root / "report.txt"
            source.write_text("data", encoding="utf-8")
            plan = build_plan(root, ".txt", "archived_")
            done = apply_plan(plan)
            self.assertEqual(len(done), 1)
            self.assertFalse(source.exists())
            self.assertEqual(
                (root / "archived_report.txt").read_text(encoding="utf-8"),
                "data",
            )

    def test_failure_rolls_back_previous_file(self) -> None:
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            first = root / "a.txt"
            second = root / "b.txt"
            first.write_text("a", encoding="utf-8")
            second.write_text("b", encoding="utf-8")
            plan = build_plan(root, ".txt", "archived_")
            real_link = os.link
            calls = 0

            def fail_second_link(src: Path, dst: Path) -> None:
                nonlocal calls
                calls += 1
                if calls == 2:
                    raise OSError("simulated link failure")
                real_link(src, dst)

            with patch("os.link", side_effect=fail_second_link):
                with self.assertRaises(OSError):
                    apply_plan(plan)

            self.assertTrue(first.exists())
            self.assertTrue(second.exists())
            self.assertFalse((root / "archived_a.txt").exists())
            self.assertFalse((root / "archived_b.txt").exists())


def run_tests() -> int:
    suite = unittest.defaultTestLoader.loadTestsFromTestCase(RenameTests)
    result = unittest.TextTestRunner(verbosity=2).run(suite)
    return 0 if result.wasSuccessful() else 1


def main() -> int:
    parser = argparse.ArgumentParser(
        description="先预览，再安全地为文件名添加前缀"
    )
    parser.add_argument("directory", nargs="?", type=Path)
    parser.add_argument("--suffix", default=".txt")
    parser.add_argument("--prefix", default="archived_")
    parser.add_argument("--apply", action="store_true", help="确认后执行重命名")
    parser.add_argument("--self-test", action="store_true", help="运行内置测试")
    args = parser.parse_args()

    if args.self_test:
        return run_tests()
    if args.directory is None:
        parser.error("请提供要处理的目录")

    try:
        plan = build_plan(args.directory, args.suffix, args.prefix)
        for source, destination in plan:
            print(f"{source.name} -> {destination.name}")
        if not plan:
            print("没有符合条件的普通文件")
            return 0
        if not args.apply:
            print(f"仅预览：共 {len(plan)} 个文件；确认后追加 --apply")
            return 0

        done = apply_plan(plan)
        print(f"完成：{len(done)} 个文件")
        return 0
    except (OSError, RuntimeError, ValueError) as exc:
        print(f"失败：{exc}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    raise SystemExit(main())
~~~

## 工程思考

预览机制的价值不在于多打印几行文字，而在于让“计划”和“副作用”分离。脚本应先把所有候选文件和目标路径算出来，再统一做校验；不要在遍历目录时发现一个文件就立即改一个文件。排序也很重要，它能让两次预览结果更容易比较，代码审查时也更容易发现异常路径。

硬链接方案不是所有环境都适用。某些网络文件系统、权限受限的目录或特殊存储不支持创建硬链接，程序会失败而不是退回到可能覆盖目标的重命名方式。硬链接和源文件位于同一目录，通常能保证同一文件系统，但这不代表它能跨盘工作。若目标目录与源目录不同，必须重新设计复制、校验和替换流程，不能直接照搬。

回滚也有边界。程序只能回滚它已经记录为完成的步骤；若系统突然断电，或清理新链接时再次失败，目录仍可能需要人工检查。对于重要资料，先复制备份或在副本上演练，比依赖一个脚本的“自动回滚”更可靠。执行前后可以保存文件清单、大小和 SHA-256 摘要，以便事后核对内容是否保持一致。

不要把符号链接当普通文件处理。链接指向的目标可能在目录之外，跟随链接进行批量操作会扩大实际影响范围。示例直接跳过符号链接，也不处理目录；如果业务确实需要递归或处理链接，应把它们作为单独选项，并为每种行为写测试。

## 升级方向

下一步可以把计划导出为 CSV，包含源路径、目标路径、文件大小和摘要；用户审查 CSV 后，再由执行阶段读取并重新校验。还可以增加 `--recursive`、排除规则、按日期生成前缀和执行日志，但每增加一种筛选条件，都应增加对应测试。

如果目录由多个进程共同写入，应在计划生成和执行期间取得目录级锁，或者将任务放到维护窗口。对于跨文件系统移动，使用“复制到临时文件、校验摘要、再原子替换目标”的独立方案，并明确处理权限、时间戳和磁盘空间。不要把单目录的简单批量改名脚本包装成通用文件迁移工具。

参考：
- [Python 3.13 pathlib：Path.rename 的平台行为](https://docs.python.org/3.13/library/pathlib.html#pathlib.Path.rename)
- [Python 3.13 os：os.link 与硬链接](https://docs.python.org/3.13/library/os.html#os.link)
- [Python 3.13 unittest.mock：patch](https://docs.python.org/3.13/library/unittest.mock.html#unittest.mock.patch)
- [Python 官方版本支持状态](https://devguide.python.org/versions/)
