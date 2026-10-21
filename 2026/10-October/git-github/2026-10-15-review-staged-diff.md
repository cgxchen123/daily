---
title: "提交前先看清改了什么：用 Git diff 降低误提交"
date: 2026-10-15
category: git-github
tags: [git, github, review]
difficulty: beginner
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 提交前先看清改了什么：用 Git diff 降低误提交

## 真实痛点

临时文件、密钥和无关修改可能被一起提交；编辑器中的文件列表不能代表暂存区内容。

## 核心原理

分别检查工作区和暂存区。先明确加入哪些路径，再审查即将提交的完整差异。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import subprocess
import sys

def git(*args: str) -> str:
    result = subprocess.run(["git", *args], text=True, capture_output=True)
    if result.returncode:
        raise RuntimeError(result.stderr.strip())
    return result.stdout

if __name__ == "__main__":
    try:
        print("状态：")
        print(git("status", "--short") or "(干净)")
        print("暂存文件：")
        print(git("diff", "--cached", "--name-only") or "(无)")
        print("暂存差异：")
        print(git("diff", "--cached"))
    except RuntimeError as exc:
        print(f"检查失败：{exc}", file=sys.stderr)
        raise SystemExit(2)
~~~

## 运行与验证

先执行 git diff --check，再用 git add 明确指定目标文件，随后运行脚本并审查输出；确认无误后才提交。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

差异检查不能证明业务逻辑正确；不要用强制推送覆盖未知远端提交。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
