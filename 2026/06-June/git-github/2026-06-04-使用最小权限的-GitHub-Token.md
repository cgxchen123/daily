---
title: "使用最小权限的 GitHub Token"
date: 2026-06-04
category: git-github
tags: [git-github, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 使用最小权限的 GitHub Token

自动化凭据只应获得完成任务所需的仓库和操作权限。优先采用短期凭据或平台提供的应用身份，避免长期使用个人高权限令牌；还要验证撤销、轮换和权限变更流程。

## 实施建议

先明确目标、输入来源和成功条件。将校验、权限判断和副作用分离，确保拒绝路径没有留下部分状态。关键行为应有可复查的测试或审计记录，并在失败时给出清楚的恢复方式。

## 示例

```bash
git status --short
# 立即撤销或轮换已泄露凭据。
# 清理历史前先协调协作者并备份。
```

## 验收清单

覆盖正常路径、边界条件和失败路径；确认日志不含秘密、资源有上限、重复操作不会造成意外副作用。示例仅展示核心机制，生产环境需要专项测试。未实际运行的测试不得宣称通过。
