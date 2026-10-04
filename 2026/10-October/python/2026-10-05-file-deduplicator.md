---
title: "别急着删文件：用 Python 做一个安全的重复文件扫描器"
date: 2026-10-05
category: python
tags: [python, hashlib, automation, file-management]
difficulty: beginner
reading_time: 8 min
python_version: "3.10+"
dependencies: []
series: null
---

# 别急着删文件：用 Python 做一个安全的重复文件扫描器

整理下载目录、照片备份或项目产物时，最危险的动作不是没找到重复文件，而是凭文件名或修改时间直接做清理。更稳妥的做法是先确认文件内容，再由人决定如何处理。

这篇文章实现一个只读的重复文件扫描器。它不会自动修改目标目录，只输出内容完全相同的文件组，适合先做盘点。

## 核心思路

文件大小不同，不可能内容完全相同，所以先按大小分组。只有大小相同的文件，才继续计算 SHA-256 摘要。摘要相同的文件归入同一组。

分层过滤比任意两两比较更实用：

1. 遍历目录中的普通文件；
2. 按文件大小分组；
3. 对同大小文件分块计算摘要；
4. 输出摘要相同的文件组。

## 完整代码

下面只使用 Python 标准库，默认 Python 3.10+。代码采用分块读取，不会把整个大文件一次性放进内存。

```python
from __future__ import annotations

import argparse
import hashlib
from pathlib import Path


CHUNK_SIZE = 1024 * 1024


def iter_files(root: Path) -> list[Path]:
    return [
        path
        for path in root.rglob("*")
        if path.is_file() and not path.is_symlink()
    ]


def file_digest(path: Path) -> str:
    digest = hashlib.sha256()

    with path.open("rb") as file:
        while chunk := file.read(CHUNK_SIZE):
            digest.update(chunk)

    return digest.hexdigest()


def group_by_size(files: list[Path]) -> dict[int, list[Path]]:
    groups: dict[int, list[Path]] = {}

    for path in files:
        groups.setdefault(path.stat().st_size, []).append(path)

    return groups


def find_duplicates(root: Path) -> list[list[Path]]:
    duplicate_groups: list[list[Path]] = []

    for candidates in group_by_size(iter_files(root)).values():
        if len(candidates) < 2:
            continue

        by_digest: dict[str, list[Path]] = {}

        for path in candidates:
            by_digest.setdefault(file_digest(path), []).append(path)

        duplicate_groups.extend(
            group for group in by_digest.values() if len(group) > 1
        )

    return duplicate_groups


def main() -> None:
    parser = argparse.ArgumentParser(
        description="查找目录中内容完全相同的文件"
    )
    parser.add_argument("root", type=Path, help="待扫描目录")
    args = parser.parse_args()

    root = args.root.expanduser().resolve()

    if not root.is_dir():
        raise SystemExit(f"目录不存在：{root}")

    groups = find_duplicates(root)

    if not groups:
        print("没有发现重复文件。")
        return

    for index, group in enumerate(groups, start=1):
        print(f"重复组 {index}：")

        for path in group:
            print(f"  {path}")

        print()


if __name__ == "__main__":
    main()
```

保存为 `file_deduplicator.py`，运行：

```bash
python file_deduplicator.py /你的/目标目录
```

## 为什么不自动清理

检测和清理不是同一个问题。程序可以确认文件内容相同，但不能替你判断：

- 哪个路径是主版本；
- 哪个文件正在被其他程序使用；
- 目录位置和文件名是否有业务意义；
- 是否存在同步、备份或权限依赖。

更安全的流程是：先扫描，再人工确认，最后单独执行可回滚的清理。

## 工程升级方向

后续可以增加排除目录、JSON/CSV 输出、单元测试和建议保留规则。但不要把“建议保留”直接变成自动删除，尤其是在备份盘和共享目录中。

## 结论

重复文件处理最容易犯的错误，是把需要证据的判断简化成“名字看起来像”。先按大小过滤，再按内容摘要确认，最后把清理决定留给人，边界更清楚，也更容易测试。
