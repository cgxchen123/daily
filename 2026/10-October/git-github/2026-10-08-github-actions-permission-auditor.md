---
title: "别让 GitHub Actions 拿到多余权限：用 Python 做一个工作流审计器"
date: 2026-10-08
category: git-github
tags: [github-actions, security, permissions, python, supply-chain]
difficulty: intermediate
reading_time: 10 min
python_version: "3.10+"
dependencies: []
series: null
---

# 别让 GitHub Actions 拿到多余权限：用 Python 做一个工作流审计器

GitHub Actions 很方便，但工作流本身就是一段会执行代码的自动化配置。

真正值得警惕的不是“用了 Actions”，而是一个本来只需要读代码的任务，最后却拿到了不必要的写权限，或者把第三方 Action 固定在一个可以被移动的标签上。

GitHub 官方文档说明，工作流可以通过 `permissions` 控制 `GITHUB_TOKEN` 的访问范围；如果显式设置某些权限，未列出的权限会被设为 `none`。官方安全指南同时建议第三方 Action 使用完整长度的 commit SHA 固定版本。

这篇不做一个“完整 YAML 安全解析器”，而是做一个更容易落地的第一道门：在提交前扫描工作流文件，发现明显的高权限和不稳定 Action 引用。

它只使用 Python 3.10+ 标准库。

## 一、先理解两个风险

### 1. 不必要的写权限

例如：

```yaml
permissions: write-all
```

或者：

```yaml
permissions:
  contents: write
```

并不是说这些配置永远错误，而是它们应该有明确理由。

一个只运行测试、读取代码和生成报告的任务，通常没有理由默认拥有仓库内容写权限。

`GITHUB_TOKEN` 是针对仓库的安装访问令牌，权限由工作流、仓库以及更高层级的设置共同决定。

### 2. 把第三方 Action 固定到可移动标签

例如：

```yaml
- uses: actions/checkout@main
```

标签和分支是可移动引用。对于第三方 Action，更稳妥的做法是使用完整 commit SHA。

所以我们可以先做一个很小的审计器，专门抓这两类问题。

## 二、完整可运行实现

保存为 `audit_workflows.py`：

```python
from __future__ import annotations

import argparse
import re
import sys
from pathlib import Path


ACTION_REF = re.compile(
    r"uses:\s*([A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+)@([^\s#]+)"
)

WRITE_PERM = re.compile(
    r"^\s{0,8}"
    r"(contents|actions|pull-requests|issues|packages|deployments|"
    r"checks|statuses|security-events|id-token)"
    r"\s*:\s*write\s*(?:#.*)?$"
)

TOP_WRITE_ALL = re.compile(
    r"^\s*permissions\s*:\s*write-all\s*$"
)


def scan_file(path: Path) -> list[str]:
    findings: list[str] = []

    try:
        lines = path.read_text(encoding="utf-8").splitlines()
    except (OSError, UnicodeDecodeError) as exc:
        return [f"{path}: 无法读取: {exc}"]

    for line_number, line in enumerate(lines, start=1):
        if TOP_WRITE_ALL.match(line):
            findings.append(
                f"{path}:{line_number}: workflow 使用 write-all"
            )

        if WRITE_PERM.match(line):
            findings.append(
                f"{path}:{line_number}: token 权限包含 write"
            )

        match = ACTION_REF.search(line)
        if match and match.group(2).lower() in {
            "main",
            "master",
            "latest",
        }:
            findings.append(
                f"{path}:{line_number}: action 未固定到不可变 SHA: "
                f"{match.group(1)}@{match.group(2)}"
            )

    return findings


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Audit GitHub Actions workflow files."
    )
    parser.add_argument(
        "root",
        help="包含 .github/workflows 的项目目录",
    )
    args = parser.parse_args()

    root = Path(args.root)

    if not root.is_dir():
        print(
            f"错误：目录不存在：{root}",
            file=sys.stderr,
        )
        return 2

    workflow_root = root / ".github" / "workflows"

    if not workflow_root.is_dir():
        print("未找到 .github/workflows，未执行扫描。")
        return 0

    findings: list[str] = []

    for path in workflow_root.rglob("*"):
        if not path.is_file():
            continue

        if path.suffix.lower() not in {".yml", ".yaml"}:
            continue

        findings.extend(scan_file(path))

    if findings:
        print("\n".join(findings))
        return 1

    print("未发现本工具定义的高风险模式。")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

运行：

```bash
python audit_workflows.py .
```

退出码：

```text
0 = 没有发现本工具定义的高风险模式
1 = 发现至少一个匹配项
2 = 输入目录错误
```

## 三、为什么故意不用 YAML 解析器

这个程序不是 YAML 安全分析器。

它只是读取工作流文本，然后检查三个明确模式：

1. `permissions: write-all`
2. 部分常见权限是否直接设置为 `write`
3. Action 是否使用 `main`、`master` 或 `latest`

这样做的优点是零依赖、容易放进 Git hook，也不需要为了一个提交前检查器再部署完整工具链。

缺点也非常明确：正则不是 YAML 解析器。

复杂的 YAML 锚点、多行结构、表达式拼接、继承关系，都可能超出这个扫描器的理解范围。

因此程序输出应该叫“发现高风险模式”，而不是“证明工作流安全”。

## 四、实际验证

这次没有只做静态检查。

在临时目录中创建了两套工作流样本。

第一套故意包含：

```yaml
permissions: write-all
```

以及：

```yaml
contents: write
```

和：

```yaml
uses: actions/checkout@main
```

实际运行返回 1，并分别报告这三个问题。

第二套改成只读权限，并把 Action 引用改成完整 SHA，实际运行返回 0。

另外测试了不存在的目录，程序返回 2。

因此验证覆盖：

- 正常通过路径；
- 高风险命中路径；
- 输入错误路径。

这里的“实际运行”仅指这个审计器在临时测试目录中的运行结果，不代表它能发现所有 GitHub Actions 安全问题。

## 五、为什么“发现 write”不能直接等于“禁止 write”

假设工作流负责自动发布版本：

```yaml
permissions:
  contents: write
```

它可能确实需要写仓库。

如果扫描器看到 `contents: write` 就一律阻断，最后开发者很可能选择关闭扫描器。

更合理的做法是：

```text
扫描器发现风险
       ↓
判断这个权限是否是业务必需
       ↓
必要 → 保留并记录原因
不必要 → 收紧权限
```

所以这个工具更适合作为审计入口，而不是一个不知道上下文就乱改配置的自动修复器。

## 六、真正值得优先做的配置

如果一个工作流只需要读取代码，可以明确写：

```yaml
permissions:
  contents: read
```

如果某个具体 job 才需要额外权限，可以把权限缩小到 job 层，而不是整个 workflow 都给高权限。

对于第三方 Action，则优先考虑：

```yaml
- uses: actions/checkout@<完整 commit SHA>
```

GitHub 也提供仓库和组织级策略，可以要求 Actions 使用完整长度 commit SHA。

## 七、这个小工具不能解决什么

它不能替代：

- GitHub Actions 官方安全策略；
- CodeQL；
- OpenSSF Scorecard；
- 完整的供应链安全审计；
- Secret scanning；
- 权限审批；
- 第三方 Action 源码审查。

尤其不要把“所有 Action 都固定 SHA”理解成“供应链风险已经解决”。

涉及 `pull_request_target`、`workflow_run` 等高权限触发器时，还需要单独考虑不可信 PR 代码的问题，不能靠这几十行正则解决。

## 八、把它接进真实项目

最简单的使用方式是把它放在 Git hook、CI 检查或者人工提交前检查里：

```text
修改 workflow
    ↓
audit_workflows.py
    ↓
没有高风险模式 → 继续
发现模式 → 人工检查
    ↓
确认权限确实需要 / 修改配置
    ↓
提交
```

下一步可以把扫描器升级成更严格的版本：

1. 使用真正的 YAML 解析器处理结构，而不是正则。
2. 支持 workflow 级和 job 级权限继承分析。
3. 检查更多可移动的 Action 引用。
4. 对完整 SHA 做来源和存在性校验。
5. 给每条发现增加严重级别和允许清单。
6. 只扫描 Git 即将提交的 workflow 文件，减少无关目录扫描。

这些升级都应该建立在真实需求上，而不是为了让代码看起来更复杂。

## 九、结论

GitHub Actions 的安全问题，很多时候不是“有没有安全工具”，而是自动化任务到底拿到了多少它根本不需要的能力。

一个几十行的 Python 审计器不能证明工作流安全，但可以把两个很容易被忽略的问题提前暴露出来：

- 不必要的写权限；
- 使用可移动标签引用第三方 Action。

真正可靠的做法不是追求一个“万能扫描器”，而是把权限、依赖版本、代码来源和触发器风险分别控制。

先把不需要的权限收掉，再把外部依赖固定住，最后才谈更复杂的安全检测。

## 参考资料

- GitHub Docs — Secure use reference
- GitHub Docs — GITHUB_TOKEN
- GitHub Docs — Workflow syntax for GitHub Actions
- GitHub Docs — Managing GitHub Actions settings
