---
title: "接口失败别无限重试：实现带上限的指数退避"
date: 2026-10-16
category: automation
tags: [automation, retry, reliability]
difficulty: intermediate
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 接口失败别无限重试：实现带上限的指数退避

## 真实痛点

立即重试会放大服务故障；无上限循环可能让定时任务永远不结束。

## 核心原理

仅对明确可重试的错误进行有限重试，逐步增加等待时间，并加入随机抖动减少同步重试。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import random
import time

class TemporaryError(Exception):
    pass

def retry(operation, attempts=4, base_delay=0.2, max_delay=3.0):
    if attempts < 1:
        raise ValueError("attempts 必须至少为 1")
    for index in range(attempts):
        try:
            return operation()
        except TemporaryError:
            if index == attempts - 1:
                raise
            ceiling = min(max_delay, base_delay * (2 ** index))
            time.sleep(random.uniform(0, ceiling))
    raise AssertionError("unreachable")

if __name__ == "__main__":
    calls = 0
    def demo():
        global calls
        calls += 1
        if calls < 3:
            raise TemporaryError("模拟临时故障")
        return "成功"
    print(retry(demo), "调用次数", calls)
~~~

## 运行与验证

保存为 retry_demo.py 并运行；示例会在两次模拟失败后成功。生产代码还应配置网络超时和总耗时预算。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

非幂等写操作重试前要确认服务端是否可能已经执行，必要时使用幂等键。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
