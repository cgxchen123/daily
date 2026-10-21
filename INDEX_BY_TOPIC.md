# 按主题索引

## Python

- 2026-01-07 — [读 CSV 时显式处理编码与错误策略](2026/01-January/python/2026-01-07-csv-encoding-detection.md)
- 2026-01-02 — [拼接文件路径前先限制输入范围](2026/01-January/python/2026-01-02-safe-path-join.md)


- 2026-10-20 — [批量处理 CSV 前先验证列名：避免静默丢数据](2026/10-October/python/2026-10-20-csv-header-validation.md)
- 2026-10-12 — [处理大文件别一次性读入内存：用 Python 流式校验 JSONL](2026/10-October/python/2026-10-12-jsonl-stream-validation.md)
- 2026-10-05 — [file deduplicator](2026/10-October/python/2026-10-05-file-deduplicator.md)

## AI Tools

- 2026-10-13 — [让模型输出 JSON 更可靠：把解析、校验和重试分开](2026/10-October/ai-tools/2026-10-13-llm-json-contract.md)
- 2026-10-06 — [ai output contract](2026/10-October/ai-tools/2026-10-06-ai-output-contract.md)

## Security

- 2026-01-08 — [文件上传必须限制大小和类型](2026/01-January/security/2026-01-08-validate-upload-size.md)
- 2026-01-03 — [日志需要排错信息，但不应该泄露密钥](2026/01-January/security/2026-01-03-redact-secrets-from-logs.md)


- 2026-10-21 — [依赖漏洞扫描结果如何处理：从告警到可追踪修复](2026/10-October/security/2026-10-21-vulnerability-triage-record.md)
- 2026-10-14 — [避免把 API 密钥写进仓库：用环境变量与启动前检查](2026/10-October/security/2026-10-14-environment-secret-check.md)
- 2026-10-07 — [secret scanner](2026/10-October/security/2026-10-07-secret-scanner.md)

## Git / GitHub

- 2026-01-04 — [撤销 Git 修改前，先分清暂存区和工作区](2026/01-January/git-github/2026-01-04-git-restore-staged-vs-worktree.md)


- 2026-10-15 — [提交前先看清改了什么：用 Git diff 降低误提交](2026/10-October/git-github/2026-10-15-review-staged-diff.md)
- 2026-10-08 — [github actions permission auditor](2026/10-October/git-github/2026-10-08-github-actions-permission-auditor.md)

## Automation

- 2026-01-10 — [调用外部命令时必须处理超时与退出码](2026/01-January/automation/2026-01-10-subprocess-timeout-and-exit-code.md)
- 2026-01-05 — [批处理任务要能安全重跑：从幂等性开始](2026/01-January/automation/2026-01-05-idempotent-file-processing.md)


- 2026-10-16 — [接口失败别无限重试：实现带上限的指数退避](2026/10-October/automation/2026-10-16-bounded-retry-backoff.md)
- 2026-10-07 — [idempotent job lock](2026/10-October/automation/2026-10-07-idempotent-job-lock.md)

## Open Source

- 2026-01-06 — [依赖版本要可复现：不要把安装结果交给运气](2026/01-January/open-source/2026-01-06-pin-dependencies.md)


- 2026-10-17 — [引入开源依赖前先做清单：版本、许可证与来源都要留证](2026/10-October/open-source/2026-10-17-dependency-inventory.md)
- 2026-10-09 — [verify release sha256](2026/10-October/open-source/2026-10-09-verify-release-sha256.md)

## Engineering

- 2026-01-09 — [健康检查要区分进程存活与服务就绪](2026/01-January/engineering/2026-01-09-health-check-vs-readiness.md)
- 2026-01-01 — [从一开始就把 Python 项目目录分清楚](2026/01-January/engineering/2026-01-01-python-project-layout.md)


- 2026-10-22 — [发布前做一份可核验清单：版本、测试、变更与回滚边界](2026/10-October/engineering/2026-10-22-release-evidence-checklist.md)
- 2026-10-18 — [让配置错误尽早暴露：在程序启动时集中验证](2026/10-October/engineering/2026-10-18-startup-config-validation.md)
- 2026-10-10 — [python cli json errors](2026/10-October/engineering/2026-10-10-python-cli-json-errors.md)

## Trends

- 2026-10-19 — [评估 RAG 别只看几个案例：建立可复现的检索测试集](2026/10-October/trends/2026-10-19-rag-retrieval-testset.md)
- 2026-10-11 — [ai coding worktree isolation](2026/10-October/trends/2026-10-11-ai-coding-worktree-isolation.md)
- 2026-10-11 — [rag retrieval offline evaluation](2026/10-October/trends/2026-10-11-rag-retrieval-offline-evaluation.md)
- 2026-10-04 — [ai agent approval gateway](2026/10-October/trends/2026-10-04-ai-agent-approval-gateway.md)
