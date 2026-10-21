---
title: "用 cherry-pick 移植修复"
date: 2026-02-24
category: git-github
tags: [git-github, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 用 cherry-pick 移植修复

cherry-pick 适合把一个明确提交移植到另一个分支，但它会创建新的提交对象，并可能引入依赖不完整的改动。执行前要阅读原提交、确认前置依赖和测试要求；出现冲突时先理解双方语义，不要只为消除冲突标记而机械选择一侧。

## 实施步骤

1. 明确输入来源、允许范围和失败行为。
2. 将校验与副作用分开，确保拒绝路径不会留下部分结果。
3. 为边界条件和故障恢复建立重复可执行的检查。
4. 保存版本、运行环境和真实输出，便于之后复查。

## 示例

```bash
git status --short
git show --stat <commit-sha>
git show <commit-sha>
# 仅在确认目标提交和影响后执行：
# git revert <commit-sha>
```

占位 SHA 必须替换为真实提交。对公共历史优先使用新的反向提交；秘密泄露还需要立即轮换凭据。

## 验收清单

覆盖正常场景、边界场景和失败场景；确认日志不含秘密，重复执行不会产生非预期副作用。本文示例不是生产级完整实现，未在目标环境实际执行的测试不得宣称通过。
