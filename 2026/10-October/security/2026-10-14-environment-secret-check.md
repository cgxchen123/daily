---
title: "避免把 API 密钥写进仓库：用环境变量与启动前检查"
date: 2026-10-14
category: security
tags: [security, secrets, python]
difficulty: beginner
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 避免把 API 密钥写进仓库：用环境变量与启动前检查

## 真实痛点

硬编码凭据可能进入 Git 历史、日志和截图；删除当前文件无法清除历史泄漏。

## 核心原理

启动时读取环境变量，缺失时快速失败；错误消息不得回显密钥值。环境变量也不是完整的秘密管理系统。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import os
import sys

def main() -> int:
    value = os.environ.get("DEMO_API_KEY", "").strip()
    if not value:
        print("配置错误：缺少 DEMO_API_KEY", file=sys.stderr)
        return 2
    print(f"凭据已加载，字符数={len(value)}；未输出凭据内容")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
~~~

## 运行与验证

PowerShell 可先设置 $env:DEMO_API_KEY='测试值'，再运行 python check_secret.py。测试值不要替换为真实凭据。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

发现真实密钥泄漏时应先撤销或轮换，再调查影响；只删除文件并不足够。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
