---
title: "读 CSV 时显式处理编码与错误策略"
date: 2026-01-07
category: python
tags: [python, csv, encoding, data-quality]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 读 CSV 时显式处理编码与错误策略

CSV 文件来自不同系统时，编码可能是 UTF-8、带 BOM 的 UTF-8 或其他本地编码。如果直接用默认参数读取，程序可能在某台机器正常、换一份文件就报错。更糟糕的是，使用忽略错误的方式解码，可能静默丢掉关键字符。

## 先明确编码约定

如果文件由你的程序生成，最好的办法是在导出协议中约定 UTF-8，并在读写两端保持一致。读取由 Excel 导出的文件时，可以根据来源明确选择是否接受 UTF-8 BOM。

```python
import csv
from pathlib import Path


def read_rows(path: Path) -> list[dict[str, str]]:
    with path.open("r", encoding="utf-8-sig", newline="") as handle:
        return list(csv.DictReader(handle))


if __name__ == "__main__":
    sample = Path("sample.csv")
    sample.write_text("name,score\nAda,10\n", encoding="utf-8-sig")
    rows = read_rows(sample)
    assert rows == [{"name": "Ada", "score": "10"}]
    sample.unlink()
    print("CSV encoding check passed")
```

保存为 `read_csv.py` 后执行。示例会创建并删除一个临时演示文件；不要把同样的删除逻辑直接用于用户指定的真实文件。

## 不要盲目“自动猜编码”

统计学方法只能给出可能性，不能可靠证明编码。若无法解码，应报告文件名、行号或处理阶段，并让调用者决定如何处理。不要默认使用 `errors="ignore"`，因为它会隐藏数据损坏。

## 生产处理建议

- 明确输入编码和分隔符。
- 用 `newline=""` 配合 `csv` 模块处理换行。
- 验证表头是否存在、是否重复、是否缺少必需列。
- 对大文件逐行处理，避免一次性载入内存。
- 对原始输入保留只读副本，错误修复应可追踪。

编码不是一个“尽量让程序读进去”的小细节，而是数据契约的一部分。无法确认时，明确失败比静默改坏数据更安全。
