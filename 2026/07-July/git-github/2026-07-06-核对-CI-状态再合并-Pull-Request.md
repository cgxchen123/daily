---
title: "核对 CI 状态再合并 Pull Request"
date: 2026-07-06
category: git-github
tags: [git-github, engineering, verification]
difficulty: intermediate
reading_time: 5 min
python_version: "3.10+"
dependencies: []
series: null
---

# 核对 CI 状态再合并 Pull Request

CI 通过只表示已配置的检查成功运行，不代表测试覆盖完整。合并前应确认所需检查来自正确提交、没有被跳过，并审查关键差异和依赖变化。若检查名称或工作流发生变化，还应同步维护分支保护规则。

## 实施步骤

1. 明确数据来源、目标范围和成功条件。
2. 在产生副作用前完成输入与权限检查。
3. 对失败、重试和恢复路径做独立验证。
4. 保存版本、命令和结果，避免把假设当作验证事实。

## 示例

```bash
git status --short
git worktree list
# 创建工作树前确认目标分支未被另一个工作树检出。
```

## 验收与限制

至少覆盖正常路径、边界条件和失败路径。示例只展示核心机制，生产系统还需根据真实环境补充测试；未执行的测试不得宣称通过。
