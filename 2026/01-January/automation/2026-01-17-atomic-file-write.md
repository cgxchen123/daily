---
title: "用临时文件和原子替换避免写出半份配置"
date: 2026-01-17
category: automation
tags: [python, filesystem, atomic-write]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用临时文件和原子替换避免写出半份配置

程序直接覆盖配置文件时，如果进程在写入中途崩溃，磁盘上可能只剩一部分内容。下次启动读取这份文件就会失败，甚至无法恢复。对同一文件系统中的普通文件，可以先写临时文件，再用替换操作切换到新版本。

## 最小实现

```python
import os
import tempfile
from pathlib import Path


def atomic_write(path: Path, content: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, temp_name = tempfile.mkstemp(dir=path.parent, prefix=f".{path.name}.")
    temp_path = Path(temp_name)
    try:
        with os.fdopen(fd, "w", encoding="utf-8", newline="") as handle:
            handle.write(content)
            handle.flush()
            os.fsync(handle.fileno())
        os.replace(temp_path, path)
    finally:
        temp_path.unlink(missing_ok=True)


if __name__ == "__main__":
    target = Path("output/config.txt")
    atomic_write(target, "mode=demo\n")
    assert target.read_text(encoding="utf-8") == "mode=demo\n"
    print("atomic write check passed")
```

保存为 `atomic_write.py` 后运行。临时文件创建在目标目录中，避免跨文件系统替换的问题；`os.replace` 在常见本地文件系统上提供原子替换语义，但完整的崩溃持久性保证仍依赖操作系统和文件系统。

## 需要明确的限制

- 并发写入仍可能发生“最后一个写入者获胜”，需要锁或版本检查。
- 原子替换不自动保留旧文件权限、ACL 或其他元数据。
- 对目录项的持久性有严格要求时，还需按平台同步目录。
- 网络文件系统和特殊存储介质可能有不同语义。

原子写入解决的是文件更新中间态问题，不是并发一致性或备份问题。关键配置仍应有校验、版本记录和可恢复副本。
