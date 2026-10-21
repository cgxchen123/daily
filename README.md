# Daily-Tech-Share

每日技术分享：面向开发者的可运行技术实践、工程经验与趋势观察。

## 仓库统计

- 总文章数：297
- 最后更新：2026-10-22
- 覆盖分类：python、ai-tools、security、git-github、automation、open-source、engineering、trends
- 分类统计：python 41、ai-tools 36、security 35、git-github 35、automation 36、open-source 34、engineering 42、trends 38
- 起始日期：2026-01-01
- 日期覆盖：2026-01-01 至 2026-10-22，共 295 个自然日；现有文章文件 297 篇。历史文件中 2026-10-07 和 2026-10-11 各有两篇，未擅自删除历史文件。

## 最新文章

- [2026-10-22 — 发布前做一份可核验清单：版本、测试、变更与回滚边界](2026/10-October/engineering/2026-10-22-release-evidence-checklist.md)
- [2026-10-21 — 依赖漏洞扫描结果如何处理：从告警到可追踪修复](2026/10-October/security/2026-10-21-vulnerability-triage-record.md)
- [2026-10-20 — 批量处理 CSV 前先验证列名：避免静默丢数据](2026/10-October/python/2026-10-20-csv-header-validation.md)
- [2026-10-19 — 评估 RAG 别只看几个案例：建立可复现的检索测试集](2026/10-October/trends/2026-10-19-rag-retrieval-testset.md)
- [2026-10-18 — 让配置错误尽早暴露：在程序启动时集中验证](2026/10-October/engineering/2026-10-18-startup-config-validation.md)
- [2026-10-17 — 引入开源依赖前先做清单：版本、许可证与来源都要留证](2026/10-October/open-source/2026-10-17-dependency-inventory.md)
- [2026-10-16 — 接口失败别无限重试：实现带上限的指数退避](2026/10-October/automation/2026-10-16-bounded-retry-backoff.md)
- [2026-10-15 — 提交前先看清改了什么：用 Git diff 降低误提交](2026/10-October/git-github/2026-10-15-review-staged-diff.md)
- [2026-10-14 — 避免把 API 密钥写进仓库：用环境变量与启动前检查](2026/10-October/security/2026-10-14-environment-secret-check.md)
- [2026-10-13 — 让模型输出 JSON 更可靠：把解析、校验和重试分开](2026/10-October/ai-tools/2026-10-13-llm-json-contract.md)
- [2026-10-12 — 处理大文件别一次性读入内存：用 Python 流式校验 JSONL](2026/10-October/python/2026-10-12-jsonl-stream-validation.md)
- [2026-10-11 — ai coding worktree isolation](2026/10-October/trends/2026-10-11-ai-coding-worktree-isolation.md)

## 按主题浏览

- [Python](INDEX_BY_TOPIC.md#python)
- [AI Tools](INDEX_BY_TOPIC.md#ai-tools)
- [Security](INDEX_BY_TOPIC.md#security)
- [Git / GitHub](INDEX_BY_TOPIC.md#git--github)
- [Automation](INDEX_BY_TOPIC.md#automation)
- [Open Source](INDEX_BY_TOPIC.md#open-source)
- [Engineering](INDEX_BY_TOPIC.md#engineering)
- [Trends](INDEX_BY_TOPIC.md#trends)

## 按日期浏览

- [2026-10-22](INDEX_BY_DATE.md#2026-10-22)
- [2026-10-21](INDEX_BY_DATE.md#2026-10-21)
- [2026-10-20](INDEX_BY_DATE.md#2026-10-20)
- [2026-10-19](INDEX_BY_DATE.md#2026-10-19)
- [2026-10-18](INDEX_BY_DATE.md#2026-10-18)
- [2026-10-17](INDEX_BY_DATE.md#2026-10-17)
- [2026-10-16](INDEX_BY_DATE.md#2026-10-16)
- [2026-10-15](INDEX_BY_DATE.md#2026-10-15)
- [2026-10-14](INDEX_BY_DATE.md#2026-10-14)
- [2026-10-13](INDEX_BY_DATE.md#2026-10-13)
- [2026-10-12](INDEX_BY_DATE.md#2026-10-12)
- [2026-10-11](INDEX_BY_DATE.md#2026-10-11)
- [2026-10-10](INDEX_BY_DATE.md#2026-10-10)
- [2026-10-09](INDEX_BY_DATE.md#2026-10-09)
- [2026-10-08](INDEX_BY_DATE.md#2026-10-08)
- [2026-10-07](INDEX_BY_DATE.md#2026-10-07)
- [2026-10-06](INDEX_BY_DATE.md#2026-10-06)
- [2026-10-05](INDEX_BY_DATE.md#2026-10-05)
- [2026-10-04](INDEX_BY_DATE.md#2026-10-04)

## 工程化机制

- [持续改进机制](CONTINUOUS_IMPROVEMENT.md)
- [改进日志](IMPROVEMENT_LOG.md)
- [发布质量门禁](QUALITY_GATE.md)
- [回归检查清单](REGRESSION_CHECKLIST.md)
- [改进记录规范](IMPROVEMENT_RECORD_SPEC.md)

## 写作原则

优先真实问题、可复现实验和可验证事实；不编造 API、数据、测试结果或项目能力。文章中的第三方事实尽量引用官方文档或一手公告。

## 目录规范

文章路径采用 YYYY/YYYY-MM-Month/category/YYYY-MM-DD-kebab-case-short-title.md。固定分类：python、ai-tools、security、git-github、automation、open-source、engineering、trends。

## 运行要求

示例代码明确标注 Python 版本和依赖。涉及外部服务时，区分本地可运行示例与接入真实服务所需的额外配置。
