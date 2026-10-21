---
title: "依赖版本要可复现：不要把安装结果交给运气"
date: 2026-01-06
category: open-source
tags: [python, dependencies, reproducibility, open-source]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 依赖版本要可复现：不要把安装结果交给运气

同一份代码在两台电脑上表现不同，常见原因之一是依赖版本不一致。若安装文件只写 `requests`，今天安装与几个月后安装可能得到不同版本，间接依赖变化也可能影响行为。

## 先区分依赖声明与锁定

项目依赖声明表达兼容范围；锁定文件记录一次解析后的具体版本集合。两者用途不同，不要把一份随手复制的版本清单当作完整的可复现机制。

简单项目可以先建立虚拟环境：

```bash
python -m venv .venv
# Windows PowerShell
.venv\\Scripts\\Activate.ps1
python -m pip install --upgrade pip
```

之后再安装项目声明的依赖并运行测试。具体使用 `requirements.txt`、锁定文件还是现代项目管理工具，应根据项目工作流统一选择。

## 生成依赖清单时的注意点

`pip freeze` 会列出当前环境安装的包版本，但它也可能把调试工具、临时依赖和与项目无关的包一起写入。它适合记录环境快照，不等于经过审核的直接依赖列表。

发布前应检查：

- Python 版本是否明确。
- 直接依赖是否有合理的版本范围。
- 锁定文件是否由项目规定的工具生成。
- CI 是否使用干净环境安装并运行测试。
- 是否有已知漏洞或许可证限制。

## 一个可执行的最小检查

```bash
python --version
python -m pip check
```

第一条记录当前解释器版本，第二条检查已安装包之间的依赖冲突。它们都不能证明整个项目没有漏洞，也不能保证不同操作系统上完全一致。

可复现构建需要明确工具版本、依赖解析结果和安装来源。不要为了追求“固定版本”就永远不升级；应在可重复测试的前提下定期更新并记录差异。
