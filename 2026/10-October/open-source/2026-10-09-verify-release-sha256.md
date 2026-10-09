---
title: "下载开源发布包后，先验 SHA-256：一个零依赖校验器"
date: 2026-10-09
category: open-source
tags: [open-source, supply-chain, sha256, python]
difficulty: beginner
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 下载开源发布包后，先验 SHA-256：一个零依赖校验器

下载开源项目的 ZIP 或安装包时，下载成功不代表文件完整。应从项目官方发布渠道取得 SHA-256，再计算本地文件摘要并比较。

GitHub 的发布资产提供 `digest` 字段，格式类似 `sha256:...`，可从官方 [Releases REST API](https://docs.github.com/en/rest/releases/releases) 查询。不要把来源不明的镜像站给出的摘要当作可信基准。

摘要的价值取决于来源。下载某个版本的 `tool.zip` 时，应在同一版本的官方 Release 页面或 API 中找到对应资产的 `digest`，同时核对版本标签和资产名称。若官方没有提供摘要，不要只对下载后的文件计算一次 SHA-256，再把结果当成独立校验；那只能记录当前文件，不能证明下载前后内容一致。

## 1. 完整实现

保存为 `verify_sha256.py`。它只使用 Python 3.10+ 标准库；文件以二进制分块读取，不会一次性载入内存。程序不负责下载文件，也不需要网络连接。

```python
from __future__ import annotations

import argparse
import hashlib
import re
import sys
from pathlib import Path


SHA256_RE = re.compile(r"^[0-9a-f]{64}$")


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as file:
        for chunk in iter(lambda: file.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Compare a local file with a trusted SHA-256 digest."
    )
    parser.add_argument("file", type=Path, help="下载到本地的文件")
    parser.add_argument("expected_sha256", help="64 位 SHA-256，可带 sha256: 前缀")
    args = parser.parse_args()

    expected = args.expected_sha256.strip().lower()
    if expected.startswith("sha256:"):
        expected = expected.removeprefix("sha256:")

    if not SHA256_RE.fullmatch(expected):
        print("错误：摘要必须是 64 位十六进制 SHA-256。", file=sys.stderr)
        return 2

    if not args.file.is_file():
        print(f"错误：文件不存在或不是普通文件：{args.file}", file=sys.stderr)
        return 2

    try:
        actual = sha256_file(args.file)
    except OSError as exc:
        print(f"错误：读取文件失败：{exc}", file=sys.stderr)
        return 2

    print(f"实际 SHA-256：{actual}")
    if actual == expected:
        print("校验通过：本地文件与给定摘要一致。")
        return 0

    print("校验失败：摘要不一致，请勿运行或安装该文件。", file=sys.stderr)
    return 1


if __name__ == "__main__":
    raise SystemExit(main())
```

## 2. 运行与验证

把 `<可信摘要>` 替换成官方发布渠道给出的值；若 API 返回 `sha256:` 前缀，程序也能识别：

```bash
python verify_sha256.py downloads/tool.zip <可信摘要>
```

退出码：`0` 表示一致，`1` 表示不一致，`2` 表示摘要格式错误、文件不存在或读取失败。这样可以让脚本、批处理或后续安装步骤根据退出码决定是否继续，而不是只看屏幕上的文字。

想先验证程序本身，可以创建一个小文件，再用 Python 计算测试摘要：

```bash
python -c "from pathlib import Path; Path('sample.bin').write_bytes(b'test data\\n')"
python -c "import hashlib; from pathlib import Path; print(hashlib.sha256(Path('sample.bin').read_bytes()).hexdigest())"
```

这只是测试程序流程，不是正式软件的可信摘要。大文件验证仍应使用上面的分块脚本，避免把整个文件读入内存。

## 3. 失败时怎么处理

- **摘要不一致：**不要运行或安装。重新从项目官方发布渠道下载，并确认版本、平台和资产名称完全对应；若重复失败，应暂停使用并向项目维护者核实。
- **摘要格式错误：**检查是否复制了完整的 64 位十六进制值，是否带有多余空格或其他文字。
- **文件不存在或无法读取：**检查路径、文件名和当前账户的读取权限。

程序按二进制读取，因此不会因为换行符或文本编码转换而改变校验逻辑。它不联网、不上传文件，也不修改下载内容。

## 4. 工程边界

摘要一致只说明本地字节与所给摘要一致。如果摘要来自被攻陷账号、恶意网页或不可信镜像，仍不能证明软件安全。优先使用项目官方渠道；高风险软件还应核验签名、发布者身份和构建来源。哈希校验是完整性检查的一环，不是对开源软件安全性的全面认证。若把校验器接入自动化安装脚本，应先校验再安装，只有退出码为 0 才能继续。不要用 `|| true` 忽略失败，也不要让旧版本摘要误用于新版本；应把版本、资产名和摘要作为一组记录。

## 参考资料

- [GitHub Changelog：Releases now expose digests for release assets](https://github.blog/changelog/2025-06-03-releases-now-expose-digests-for-release-assets/)
- [GitHub REST API：Releases](https://docs.github.com/en/rest/releases/releases)
- [Python 3.10 文档：hashlib](https://docs.python.org/3.10/library/hashlib.html)
