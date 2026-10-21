---
title: "调用外部命令时必须处理超时与退出码"
date: 2026-01-10
category: automation
tags: [python, subprocess, automation, reliability]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 调用外部命令时必须处理超时与退出码

自动化脚本常常需要调用 Git、编译器或系统工具。直接启动子进程并假设它总会成功，会让失败变成难以定位的挂起任务。外部命令可能卡住、返回非零退出码，或者产生过多输出。

## 使用 subprocess.run 设置超时

```python
import subprocess


def run_check(command: list[str]) -> str:
    result = subprocess.run(
        command,
        check=True,
        capture_output=True,
        text=True,
        timeout=10,
    )
    return result.stdout


if __name__ == "__main__":
    print(run_check(["python", "--version"]).strip())
```

保存为 `run_command.py` 后执行。这里使用参数列表而不是拼接 shell 字符串，避免不必要的 shell 解释；`check=True` 会在退出码非零时抛出 `CalledProcessError`，超时则抛出 `TimeoutExpired`。

## 错误处理应保留上下文

调用方应捕获并区分超时、命令不存在、非零退出码和输出解析失败。不要把所有异常统一吞掉后返回“成功”，也不要在日志中无条件打印可能含敏感信息的完整参数。

## 关于安全边界

参数列表能降低 shell 注入风险，但如果程序把不可信输入直接作为命令参数，目标程序本身仍可能把它解释为特殊选项或路径。应使用允许命令白名单、验证参数格式、限制执行时长和权限，并避免以高权限账户运行。

## 自动化任务的验收清单

- 命令成功时能读取结果。
- 非零退出码会让任务失败并留下可诊断信息。
- 超时后能终止或清理相关进程。
- 输入路径和参数经过验证。
- 日志不包含令牌、密码或不必要的个人数据。

外部命令不是普通函数调用：它有独立的进程、退出状态、资源消耗和权限边界。把这些边界显式处理，脚本才适合进入定时任务。
