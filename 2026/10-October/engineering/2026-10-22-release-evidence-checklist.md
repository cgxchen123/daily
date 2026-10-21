---
title: "发布前做一份可核验清单：版本、测试、变更与回滚边界"
date: 2026-10-22
category: engineering
tags: [engineering, release, checklist]
difficulty: intermediate
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 发布前做一份可核验清单：版本、测试、变更与回滚边界

## 真实痛点

发布流程依赖个人记忆，测试、配置迁移和回滚准备容易漏项；“测试通过”也常缺少实际证据。

## 核心原理

将检查项、状态、执行命令和证据绑定，区分已执行、失败、未执行与不适用。清单本身不执行测试，不能冒充测试报告。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import json
import sys
from pathlib import Path

REQUIRED = {"version", "commit", "owner", "tests", "migration", "rollback_plan"}

def main() -> int:
    path = Path(sys.argv[1] if len(sys.argv) > 1 else "release.json")
    try:
        data = json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError) as exc:
        print(f"无法读取发布记录: {exc}", file=sys.stderr)
        return 2
    if not isinstance(data, dict) or REQUIRED - data.keys():
        print("发布记录缺少必需字段", file=sys.stderr)
        return 1
    if not isinstance(data["tests"], list) or not data["tests"]:
        print("tests 必须是非空数组", file=sys.stderr)
        return 1
    bad = [x for x in data["tests"] if not isinstance(x, dict)
           or x.get("status") not in {"passed", "failed", "not_run", "n_a"}
           or not x.get("evidence")]
    if bad or any(x["status"] in {"failed", "not_run"} for x in data["tests"]):
        print("发布门禁未通过：检查状态和证据", file=sys.stderr)
        return 1
    print("记录结构完整；仍需人工核实证据真实性")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
~~~

## 运行与验证

建立 release.json，tests 中每项包含 status 和 evidence，再运行 python release_check.py。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

该脚本只验证记录格式，不运行测试、不验证 commit，也不执行部署或回滚。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
