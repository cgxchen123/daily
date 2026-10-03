# 贡献指南

文章按年份、月份和固定主题分类归档。请确保代码可运行，不提交密钥或个人敏感信息。

## 发布规范

1. 新文章必须遵循 `YYYY/YYYY-MM-Month/category/YYYY-MM-DD-kebab-case-short-title.md`。
2. 固定分类仅限：python、ai-tools、security、git-github、automation、open-source、engineering、trends。
3. 发布前必须读取最新的 `main`，检查当天是否已有文章，避免重复或覆盖历史内容。
4. 多文件发布优先使用 Git Tree + Commit 组成一次原子提交，再以非强制快进方式更新默认分支；这是正常的 Git 数据写入方式，不用于绕过任何安全控制。
5. 如果写入出现 401/403/404、权限异常、冲突或其他非预期失败，应立即停止，不得静默重试或切换成另一条写入路径规避错误。
6. 已发布历史文章默认只读；除非任务明确要求修复并先完成内容审计，否则不得修改、覆盖或删除历史文章。
