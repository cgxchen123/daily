---
title: "拼接文件路径前先限制输入范围"
date: 2026-01-02
category: python
tags: [python, pathlib, filesystem, security]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 拼接文件路径前先限制输入范围

文件路径来自用户输入、CSV 或接口参数时，直接把它拼到目标目录后面并不安全。像 `../../outside.txt` 这样的相对路径可能跳出预期目录，导致程序读取或覆盖不该访问的文件。只检查字符串是否包含两个点也不可靠，因为路径有不同分隔符和规范化形式。

## 用 pathlib 做目录边界检查

```python
from pathlib import Path


def resolve_under(root: Path, user_path: str) -> Path:
    base = root.resolve()
    candidate = (base / user_path).resolve()
    if candidate != base and base not in candidate.parents:
        raise ValueError("path escapes the allowed directory")
    return candidate


if __name__ == "__main__":
    root = Path("sandbox")
    print(resolve_under(root, "reports/today.txt"))
    try:
        resolve_under(root, "../secret.txt")
    except ValueError:
        print("blocked traversal")
    else:
        raise SystemExit("unsafe path was accepted")
```

保存为 `safe_path.py`，运行 `python safe_path.py`。示例检查规范化后的路径是否仍位于根目录中。运行前不需要创建文件，因为这段代码只解析路径。

## 不能忽略的边界

路径检查不等于完整的文件系统沙箱。如果攻击者能在检查与打开文件之间替换符号链接，就可能形成竞态条件；高风险服务需要更严格的文件描述符相对操作、权限隔离和操作系统沙箱。还要明确是否允许目标目录本身、绝对路径和符号链接。

## 上线前检查

1. 测试普通子目录、空字符串、绝对路径和父目录穿越。
2. 明确路径是用于读取还是写入，写入操作要更谨慎。
3. 对允许访问的根目录使用固定配置，不要让请求方自行指定根目录。
4. 日志记录拒绝原因，但避免泄露服务器敏感路径。

安全的原则是：先定义可访问边界，再解析路径并验证边界，不能只靠字符串过滤。
