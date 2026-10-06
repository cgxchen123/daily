---
title: "别等密钥泄漏后再处理：用 Python 做一个提交前的敏感信息扫描器"
date: 2026-10-07
category: security
tags: [secret-scanning, security, python, git]
difficulty: intermediate
reading_time: 10 min
python_version: "3.10+"
dependencies: []
series: null
---

# 别等密钥泄漏后再处理：用 Python 做一个提交前的敏感信息扫描器

很多敏感信息泄漏问题并不发生在服务器，而是发生在开发者准备提交代码的那几分钟。

一个配置文件、备份文件、调试日志或者临时脚本，可能把密码、访问令牌、凭据片段一起带进 Git。真正麻烦的地方不是“有没有一个扫描器”，而是**扫描应该尽量靠近数据离开本机的那个时间点**。

这篇不做“万能密钥识别器”。我们只做一个边界清楚、能本地运行的版本：

- 扫描当前目录下的常见文本文件；
- 识别几类高风险命名和凭据格式；
- 跳过 .git、二进制文件和明显的构建目录；
- 输出文件、行号和匹配类型；
- 发现风险时返回非 0 退出码，方便接进提交前检查；
- 不读取或上传文件到任何外部服务。

## 一、为什么不要只依赖“别把密钥提交上去”

口头规范解决不了所有误操作。

例如下面这些文件很容易被忽略：

```text
.env
config.local.json
debug.log
backup/config.old
scripts/test-login.py
```

而且风险并不只来自明显的 password=。私钥块、带 token / secret / api_key 等命名的配置项，也值得进入人工复核。

因此一个实用的第一层防线应该遵循一个原则：

> **宁可报告“疑似敏感信息”，也不要把扫描器包装成“能证明没有泄漏”。**

正则匹配只能发现符合规则的文本，不能证明内容一定是凭据，也不能证明没有漏检。

## 二、扫描器应该放在哪里

推荐的最小链路是：

```text
工作区
  ↓
排除明显无关目录/二进制
  ↓
逐文件读取文本
  ↓
规则匹配
  ↓
输出文件 + 行号 + 类型
  ↓
无命中：退出 0
有命中：退出 1
  ↓
交给人工复核或后续门禁
```

这里故意没有加入“自动删除”“自动上传”“自动撤销凭据”等动作。

因为扫描器负责发现问题，修复和权限处置应该由另一层系统完成。

## 三、一个只用 Python 标准库的实现

下面的程序保存成 secret_scan.py 即可运行。它只使用 Python 3.10+ 标准库，不需要 pip install。

```python
from __future__ import annotations

import argparse
import re
import sys
from pathlib import Path


SKIP_DIRS = {
    ".git",
    ".venv",
    "venv",
    "node_modules",
    "__pycache__",
    "dist",
    "build",
}

PRIVATE_HEADER = "-----BEGIN " + "PRIVATE KEY-----"

RULES = [
    ("private-key", re.compile(re.escape(PRIVATE_HEADER))),
    (
        "assignment-secret",
        re.compile(
            r"(?i)\b(?:password|passwd|secret|api[_-]?key|access[_-]?token)"
            r"\s*[:=]\s*[\"']?[^\"'\s]{8,}"
        ),
    ),
    (
        "bearer-token",
        re.compile(
            r"(?i)\bAuthorization\s*:\s*Bearer\s+[A-Za-z0-9._~+/=-]{12,}"
        ),
    ),
]


def is_binary(path: Path) -> bool:
    try:
        sample = path.read_bytes()[:4096]
    except OSError:
        return True
    return b"\x00" in sample


def iter_files(root: Path):
    for path in root.rglob("*"):
        if not path.is_file():
            continue
        if any(part in SKIP_DIRS for part in path.parts):
            continue
        yield path


def scan_file(path: Path):
    try:
        text = path.read_text(encoding="utf-8")
    except (UnicodeDecodeError, OSError):
        return []

    findings = []
    for line_number, line in enumerate(text.splitlines(), start=1):
        for rule_name, pattern in RULES:
            if pattern.search(line):
                findings.append((line_number, rule_name))
    return findings


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Scan a workspace for suspicious secrets."
    )
    parser.add_argument("root", nargs="?", default=".", help="Directory to scan")
    args = parser.parse_args()

    root = Path(args.root).resolve()
    if not root.is_dir():
        print(f"错误：目录不存在：{root}", file=sys.stderr)
        return 2

    total = 0
    for path in iter_files(root):
        if is_binary(path):
            continue

        for line_number, rule_name in scan_file(path):
            relative = path.relative_to(root)
            print(f"{relative}:{line_number}: {rule_name}")
            total += 1

    if total:
        print(f"\n发现 {total} 个疑似敏感信息位置，请人工复核。", file=sys.stderr)
        return 1

    print("未发现匹配规则。")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

运行：

```bash
python secret_scan.py .
```

没有命中时返回 0；有命中时返回 1；参数或目录错误返回 2。

## 四、为什么要把“疑似”写进程序设计

假设文件里出现：

```text
password = "example-password"
```

扫描器可以发现它，但无法知道这个字符串是不是生产密码。

反过来，一个真正的秘密也可能：

- 使用了完全不同的变量名；
- 被拆成多个字符串；
- 存在二进制或压缩文件里；
- 使用了扫描规则没有覆盖的凭据格式。

所以这个工具的正确定位是：

**提交前的低成本发现层，而不是安全证明工具。**

这也决定了它的输出不应该打印完整秘密。当前版本只输出文件、行号和规则名称，不把匹配内容复制到终端日志中。

## 五、把它接到 Git 提交前检查

如果团队已经有 Git hooks，可以让提交动作先调用这个扫描器：

```bash
python secret_scan.py .
```

然后根据退出码决定是否继续后续流程。

一个重要的工程原则是：**不要把扫描器写成“发现风险就自动修改所有文件”的脚本。**

自动改写可能造成：

- 误删正常配置；
- 修改不应该修改的历史文件；
- 让开发者误以为问题已经被修复；
- 把敏感信息重新写入另一个日志或缓存。

发现、确认、撤销、替换、历史清理应该是不同步骤。

## 六、这类扫描器最容易犯的几个错误

### 1. 只扫描扩展名

只扫描 .py、.js、.json 会漏掉 .env、无扩展名配置文件和日志。

### 2. 直接把完整匹配内容打印出来

这相当于发现秘密后又把秘密复制到 CI 日志、终端历史或聊天记录。

### 3. 只检查当前文件，不考虑目录

.git、虚拟环境、依赖目录和构建产物会制造大量噪声，所以需要明确排除策略。

### 4. 把“0 命中”解释成“绝对安全”

扫描结果只能说明“当前规则没有发现匹配项”。

### 5. 无限增加正则

规则越多不一定越安全。误报太高，开发者最终会选择关闭检查。

## 七、下一步怎么升级

第一步，可以把规则拆成独立配置，并为每条规则增加测试样本：至少包含一个应该命中的样本和一个不应该命中的样本。

第二步，把扫描范围改成“Git 即将提交的文件”，而不是整个工作区。这样速度更快，也更接近真正的提交门禁。

第三步，为高风险规则增加严重级别，例如：

```text
critical
high
medium
low
```

再根据团队规则决定哪些级别阻断提交。

第四步，如果项目确实需要更强的凭据检测，再考虑成熟的专用 secret-scanning 工具。不要因为一个几十行脚本能跑，就把它宣传成完整的凭据防护系统。

## 结论

安全检查最有价值的时机，往往不是事故发生之后，而是数据准备离开开发环境之前。

一个标准库实现的小扫描器并不能解决全部秘密泄漏问题，但它可以提供一条便宜、透明、可审计的第一道防线。

真正重要的不是“正则写得多复杂”，而是把边界讲清楚：

**扫描器负责发现疑似风险；人工或专门的安全系统负责确认；凭据撤销和历史清理负责处置。**

把这三件事分开，系统才不会因为一个简单脚本“看起来能扫”就产生虚假的安全感。
