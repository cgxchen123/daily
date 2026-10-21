---
title: "用 diff check 发现空白错误"
date: 2026-01-23
category: git-github
tags: [git-github, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 diff check 发现空白错误

提交前审查差异比只看文件列表更可靠。git diff --check 可以发现部分空白字符问题，但不能判断业务逻辑是否正确；还要分别查看工作区和暂存区，避免把未审查的文件一起提交。

## 示例与验证

保存示例并在对应环境中执行；Git 命令应在目标仓库目录执行。

```bash
git status --short
git diff --check
git diff
git diff --staged

# 确认差异后再提交
# git commit -m "fix: describe the behavior change"
```

## 验收与限制

验证正常输入、边界输入和失败路径；确认没有意外副作用，并记录真实运行环境与输出。示例不等于生产级实现，涉及并发、权限、网络或真实数据时应增加针对性测试。未实际执行的检查不得标记为通过。
