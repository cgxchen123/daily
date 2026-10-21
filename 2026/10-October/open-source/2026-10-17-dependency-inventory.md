---
title: "引入开源依赖前先做清单：版本、许可证与来源都要留证"
date: 2026-10-17
category: open-source
tags: [open-source, license, dependencies]
difficulty: intermediate
reading_time: 6
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 引入开源依赖前先做清单：版本、许可证与来源都要留证

## 真实痛点

包能安装不代表它适合发布。版本、许可证、来源和维护状态缺少记录，会增加合规与供应链风险。

## 核心原理

记录组件名、精确版本、来源、许可证证据、用途与复核日期。自动扫描是辅助，不能替代许可证判断。

## 完整代码

Python 3.10+，仅使用标准库。保存为对应的 .py 文件：

~~~python
import csv
from pathlib import Path

FIELDS = ["name", "version", "source", "license", "evidence", "purpose"]

def check(path: str) -> list[str]:
    errors = []
    with Path(path).open(newline="", encoding="utf-8") as stream:
        reader = csv.DictReader(stream)
        if reader.fieldnames != FIELDS:
            return ["CSV 表头不符合要求"]
        for line, row in enumerate(reader, 2):
            for field in FIELDS:
                if not (row.get(field) or "").strip():
                    errors.append(f"第 {line} 行缺少 {field}")
            if row.get("source", "").strip() and not row["source"].startswith("https://"):
                errors.append(f"第 {line} 行 source 不是 HTTPS 地址")
    return errors

if __name__ == "__main__":
    issues = check("dependencies.csv")
    print("\n".join(issues) if issues else "字段完整；许可证仍需人工核验")
~~~

## 运行与验证

创建 dependencies.csv，表头按代码中的 FIELDS 设置，然后运行脚本检查清单完整性。

执行时要检查实际输出和退出码；本文未声称在目标读者的机器上运行过。建议再补充一个正常输入、一个边界输入和一个错误输入，确保失败路径也能被观察到。

## 工程思考

把输入校验、业务规则和外部副作用分开，错误消息应指出字段或行号，但不应泄漏凭据或完整敏感数据。自动化程序应使用明确退出码，让上游任务能可靠地区分成功、输入错误和环境故障。对文件写入、远端更新等有副作用的操作，应先确认目标和权限，并保留可追溯证据。

## 升级方向与局限

字段完整不代表许可证兼容；还要核对具体发行版本、子依赖、字体、模型权重和打包资源。
进一步可为核心函数添加单元测试，记录依赖和运行环境，并把真实测试结果与文档说明分开维护。示例用于说明一种可复现的工程模式，不应被理解为完整生产级工具。
