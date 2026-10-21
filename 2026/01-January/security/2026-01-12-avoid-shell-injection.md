---
title: "避免命令注入：不要把用户输入拼进 shell 字符串"
date: 2026-01-12
category: security
tags: [security, subprocess, command-injection]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 避免命令注入：不要把用户输入拼进 shell 字符串

当 Python 需要调用外部程序时，直接把用户输入拼接成 shell 命令字符串，会让输入中的特殊字符被 shell 重新解释。例如文件名里出现分号、管道或命令替换语法，可能使程序执行预期之外的操作。

## 优先使用参数列表

```python
import subprocess


def git_status(repo_path: str) -> str:
    result = subprocess.run(
        ["git", "-C", repo_path, "status", "--short"],
        shell=False,
        check=True,
        capture_output=True,
        text=True,
        timeout=10,
    )
    return result.stdout


if __name__ == "__main__":
    print(git_status("."))
```

保存为 `status.py`，在安装了 Git 的目录运行。这里把每个参数单独传递，避免 shell 解析整条字符串。路径仍应来自可信配置或经过校验；参数列表不是让所有不可信输入都自动安全。

## 还要控制目标程序

即使关闭 shell，目标程序仍可能把以连字符开头的输入当成选项。应使用明确的参数位置、必要时使用目标程序支持的 `--` 终止选项解析，并限制允许执行的命令和工作目录。

## 验收要点

- 不调用 `shell=True` 来拼接动态输入。
- 设定超时并检查非零退出码。
- 不以管理员或 root 权限执行普通任务。
- 日志记录命令类别和结果，不泄露令牌或完整敏感参数。
- 为异常输入、路径空格、非零退出码和超时编写测试。

命令注入防护应从接口设计开始：固定命令、分离参数、校验输入并限制权限。不要把“我加了引号”当成安全证明。
