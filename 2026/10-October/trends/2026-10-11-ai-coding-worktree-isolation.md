---
title: "AI 编码任务并行时，别共用一个工作目录：用 Git worktree 做隔离"
date: 2026-10-11
category: trends
tags: [ai-coding-agents, git-worktree, parallel-development, isolation, python]
difficulty: intermediate
reading_time: 10 min
python_version: "3.10+"
dependencies: []
series: null
---

# AI 编码任务并行时，别共用一个工作目录：用 Git worktree 做隔离

## 真实痛点：两个任务同时改一个目录，问题不只是代码冲突

当你让两个 AI 编码任务并行处理不同问题时，如果它们共用同一个工作目录，风险往往早于 Git 冲突出现：一个任务切换分支，另一个任务的文件状态跟着变化；一个任务执行 `git add`，可能把另一个任务的修改一起放进暂存区；某个代理还没完成，另一个代理就开始运行测试，测试结果因此无法对应到明确的代码版本。

把任务写进不同提示词，并不能隔离文件系统和 Git 索引。更稳妥的办法是：**每个并行任务使用独立的工作目录和分支，完成后再通过正常的代码审查与合并流程整合。**

Git 自带的 `git worktree` 可以让一个仓库同时拥有多个工作树。它们共享仓库对象，但各自有独立的工作目录、索引和 `HEAD`。官方文档也说明，默认情况下，同一个本地分支不能同时在多个工作树中检出；不要为了强行复用分支而随意加 `--force`。参考 [Git 官方 worktree 文档](https://git-scm.com/docs/git-worktree)。

本文实现一个本地小工具 `worktree_guard.py`，用于规划或创建隔离工作区。它遵循几个保守原则：

- 默认只输出计划，不修改仓库；必须显式加 `--apply` 才创建。
- 每个任务创建独立的 `agent/<任务名>` 分支。
- 目标目录或分支已存在时直接拒绝，不覆盖、不强制复用。
- 不提供自动删除命令，避免误删仍有未提交工作的目录。
- 使用参数数组调用 Git，不通过 Shell 拼接执行命令。

## 核心原理：隔离工作树，不等于复制一份完整仓库

`git worktree add` 会为同一仓库增加一个关联工作树。这样做通常比为每个任务重新克隆仓库更轻便，也能让每个任务拥有自己的检出目录和索引状态。不过，它不是虚拟机或安全沙箱：仓库对象和部分配置仍然共享，工作树之间也不会自动解决最终的代码合并冲突。

本工具把创建流程分成四步：

1. 找到当前仓库根目录，避免从子目录运行时误判位置。
2. 校验任务名称、工作区路径、分支名和基点提交。
3. 先检查目标路径与本地分支是否已存在。
4. 默认输出命令计划；只有使用 `--apply` 才调用 `git worktree add`，随后再读取 `git worktree list --porcelain` 验证结果。

这里刻意不实现自动清理。工作区完成后，应先检查未提交改动、确认任务结果已保存，再由人决定是否执行 `git worktree remove <路径>`。不要把“任务结束”直接等同于“目录可以删除”。

## 完整可运行代码

保存为 `worktree_guard.py`。要求 Python 3.10+，只使用标准库；此外需要本机已安装 Git，并且当前目录位于一个有效的 Git 工作树中。脚本不会联网，也不会访问模型 API。

```python
#!/usr/bin/env python3
"""Safely plan or create an isolated Git worktree for a parallel coding task."""

from __future__ import annotations

import argparse
import json
import re
import subprocess
import sys
from pathlib import Path
from typing import Any

NAME_PATTERN = re.compile(r"^[a-z0-9][a-z0-9._-]{0,39}$")


class CommandFailure(RuntimeError):
    pass


def run_git(repo: Path, *args: str) -> subprocess.CompletedProcess[str]:
    try:
        return subprocess.run(
            ["git", *args],
            cwd=repo,
            text=True,
            encoding="utf-8",
            errors="replace",
            capture_output=True,
            check=False,
        )
    except FileNotFoundError as exc:
        raise CommandFailure("找不到 git，请先安装 Git 并确认它在 PATH 中。") from exc


def require_success(result: subprocess.CompletedProcess[str], command: str) -> str:
    if result.returncode != 0:
        detail = result.stderr.strip() or result.stdout.strip() or "无额外错误信息"
        raise CommandFailure(f"{command} 失败（退出码 {result.returncode}）：{detail}")
    return result.stdout.strip()


def find_repo_root() -> Path:
    try:
        result = subprocess.run(
            ["git", "rev-parse", "--show-toplevel"],
            text=True,
            encoding="utf-8",
            errors="replace",
            capture_output=True,
            check=False,
        )
    except FileNotFoundError as exc:
        raise CommandFailure("找不到 git，请先安装 Git 并确认它在 PATH 中。") from exc
    output = require_success(result, "git rev-parse --show-toplevel")
    return Path(output).resolve()


def parse_worktree_list(output: str) -> list[dict[str, Any]]:
    records: list[dict[str, Any]] = []
    current: dict[str, Any] = {}

    def finish() -> None:
        nonlocal current
        if current:
            records.append(current)
            current = {}

    for line in output.splitlines():
        if not line:
            finish()
        elif line.startswith("worktree "):
            current["path"] = line[len("worktree "):]
        elif line.startswith("HEAD "):
            current["head"] = line[len("HEAD "):]
        elif line.startswith("branch "):
            branch = line[len("branch "):]
            current["branch"] = branch.removeprefix("refs/heads/")
        elif line == "detached":
            current["detached"] = True
        elif line == "bare":
            current["bare"] = True
        elif line.startswith("locked"):
            current["locked"] = True
        elif line.startswith("prunable"):
            current["prunable"] = True
    finish()
    return records


def list_worktrees(repo: Path) -> int:
    output = require_success(
        run_git(repo, "worktree", "list", "--porcelain"),
        "git worktree list --porcelain",
    )
    print(json.dumps(parse_worktree_list(output), ensure_ascii=False, indent=2))
    return 0


def create_worktree(repo: Path, args: argparse.Namespace) -> int:
    name = args.name
    if not NAME_PATTERN.fullmatch(name):
        raise CommandFailure(
            "名称只能由小写字母、数字、点、下划线和连字符组成，"
            "且必须以字母或数字开头，长度为 1–40。"
        )

    if args.workspace_root is None:
        workspace_root = (repo.parent / f"{repo.name}-worktrees").resolve()
    else:
        workspace_root = args.workspace_root.expanduser().resolve()

    try:
        workspace_root.relative_to(repo)
    except ValueError:
        pass
    else:
        raise CommandFailure("工作区根目录不能放在当前仓库内部，请改用仓库外的目录。")

    if workspace_root.exists() and not workspace_root.is_dir():
        raise CommandFailure(f"工作区根路径不是目录：{workspace_root}")

    target = workspace_root / name
    if target.exists() or target.is_symlink():
        raise CommandFailure(f"目标路径已存在，拒绝覆盖：{target}")

    branch = f"agent/{name}"
    format_check = run_git(repo, "check-ref-format", "--branch", branch)
    if format_check.returncode != 0:
        raise CommandFailure(f"Git 不接受生成的分支名：{branch}")
    branch_check = run_git(repo, "show-ref", "--verify", "--quiet", f"refs/heads/{branch}")
    if branch_check.returncode == 0:
        raise CommandFailure(f"本地分支已存在，拒绝复用或覆盖：{branch}")
    if branch_check.returncode != 1:
        detail = branch_check.stderr.strip() or f"退出码 {branch_check.returncode}"
        raise CommandFailure(f"检查分支是否存在失败：{detail}")

    if args.base.startswith("-"):
        raise CommandFailure("--base 不能以 '-' 开头。")
    base_result = run_git(repo, "rev-parse", "--verify", "--quiet", f"{args.base}^{{commit}}")
    base_sha = require_success(base_result, f"校验基点 {args.base!r}")

    command = ["git", "worktree", "add", "-b", branch, str(target), base_sha]
    if not args.apply:
        print(json.dumps({
            "mode": "dry-run",
            "repository": str(repo),
            "branch": branch,
            "path": str(target),
            "base_sha": base_sha,
            "command": command,
            "note": "仅生成计划；添加 --apply 才会实际创建工作区和分支。",
        }, ensure_ascii=False, indent=2))
        return 0

    workspace_root.mkdir(parents=True, exist_ok=True)
    result = run_git(repo, "worktree", "add", "-b", branch, str(target), base_sha)
    require_success(result, "git worktree add")
    records = parse_worktree_list(require_success(
        run_git(repo, "worktree", "list", "--porcelain"),
        "git worktree list --porcelain（发布后验收）",
    ))
    if not any(
        Path(item.get("path", "")).resolve() == target.resolve()
        and item.get("branch") == branch
        for item in records
    ):
        raise CommandFailure("Git 命令返回成功，但回读清单未找到目标工作区；请人工检查 git worktree list。")

    print(json.dumps({
        "mode": "created",
        "repository": str(repo),
        "branch": branch,
        "path": str(target),
        "base_sha": base_sha,
        "verified": True,
    }, ensure_ascii=False, indent=2))
    return 0


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description="为并行编码任务安全规划或创建独立 Git worktree。"
    )
    subparsers = parser.add_subparsers(dest="command", required=True)

    new_parser = subparsers.add_parser("new", help="规划或创建一个新的隔离工作区")
    new_parser.add_argument("name", help="任务名称，例如 issue-42")
    new_parser.add_argument("--base", default="HEAD", help="起始提交或引用，默认 HEAD")
    new_parser.add_argument(
        "--workspace-root",
        type=Path,
        help="工作区根目录；默认是当前仓库同级的 <仓库名>-worktrees",
    )
    new_parser.add_argument(
        "--apply",
        action="store_true",
        help="实际创建分支和工作区；省略时只输出计划",
    )

    subparsers.add_parser("list", help="以 JSON 格式列出 Git 已登记的工作区")
    return parser


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    try:
        repo = find_repo_root()
        if args.command == "list":
            return list_worktrees(repo)
        return create_worktree(repo, args)
    except CommandFailure as exc:
        print(f"错误：{exc}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    raise SystemExit(main())

```

## 立即运行

在现有 Git 仓库的任意子目录中保存脚本后，先查看已登记的工作树：

```bash
python worktree_guard.py list
```

为名为 `issue-42` 的任务生成计划。此命令不会创建目录或分支：

```bash
python worktree_guard.py new issue-42
```

确认计划里的仓库、路径和基点都正确后，再显式执行：

```bash
python worktree_guard.py new issue-42 --apply
```

默认会把工作区放到当前仓库同级的 `<仓库名>-worktrees/issue-42` 目录，并创建 `agent/issue-42` 分支。完成后重新列出工作区：

```bash
python worktree_guard.py list
```

如果希望指定位置，可以使用 `--workspace-root`，例如：

```bash
python worktree_guard.py new issue-42 \
  --workspace-root /work/agent-worktrees
```

这条命令仍然只生成计划；若确认无误，再补上 `--apply`。脚本会拒绝把工作区根目录放进当前仓库内部，以免工作树内容被误认为主工作树中的普通文件。

## 工程思考：让每个任务的输入和输出都能被追溯

隔离工作目录的价值不是让多个代理“自动协作”，而是减少彼此意外影响的机会。建议给每个任务固定记录：

- 任务目标与验收条件；
- 使用的分支名和起始提交 SHA；
- 修改了哪些文件；
- 执行了哪些测试及真实退出码；
- 是否存在未提交或未跟踪文件；
- 最终由谁审查、合并或关闭任务。

还要注意三个边界：

1. **同一分支不能默认多处同时检出。** 本工具为每个新任务创建新分支；若目标分支已经存在，会停止，而不是偷偷改用其他分支。
2. **工作树不是权限隔离。** 如果代理不可信，不应只靠 `worktree` 防止它读取凭据、访问网络或修改工作树之外的文件；还需要容器、虚拟机、最小权限和凭据隔离。
3. **合并仍然需要审查。** 两个任务即使在独立工作树中各自通过测试，合并后仍可能出现接口不兼容、行为冲突或测试失效。应在整合后的目标分支上重新运行相关测试。

本次在临时本地 Git 仓库中进行了实际验证：列出现有工作树、默认 dry-run、显式创建分支与工作树、创建后回读、重复目标拒绝、非法路径名称拒绝、非法 Git 分支名拒绝、无效基点拒绝，以及确认主工作树保持干净。测试环境为 Python 3.13.5、Git 2.47.3；上述检查均通过。这些结果只证明示例在该临时测试场景中的行为，不代表它已覆盖所有操作系统、Git 版本或复杂仓库配置。

## 升级方向

下一步可以为这个工具补充自动化测试，覆盖 Windows 路径、符号链接、已存在远程分支、子模块和多个并发创建请求等边界情况；也可以在任务清单中记录每个工作树对应的 issue、起始 SHA、测试结果和审查状态。

不要急着添加自动删除、强制覆盖或自动合并功能。对于并行编码任务，更值得优先保证的是：创建行为可预测、失败时不破坏现有工作、结果能被回读验证，以及每个任务在合并前都有明确的验收责任人。

## 参考资料

- [Git 官方文档：git-worktree](https://git-scm.com/docs/git-worktree)
- [Git 官方文档：git-check-ref-format](https://git-scm.com/docs/git-check-ref-format)
