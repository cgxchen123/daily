---
title: "给 API 令牌最小权限，并建立轮换和撤销流程"
date: 2026-01-19
category: security
tags: [security, api-token, least-privilege]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 给 API 令牌最小权限，并建立轮换和撤销流程

长期有效且权限过大的 API 令牌，一旦泄露，影响范围可能远大于单个应用。把令牌藏进环境变量只是避免硬编码的一步；还需要限制权限、缩短有效期、控制存储位置，并验证撤销流程真的有效。

## 权限设计

- 为不同应用和环境使用不同令牌。
- 只授予完成任务所需的仓库、资源和操作权限。
- 优先使用短期凭据或平台支持的工作负载身份。
- 避免把个人访问令牌用于长期自动化。
- 在日志、错误报告和截图中对凭据做脱敏。

## 启动时检查配置是否存在

```python
import os


def require_token(name: str = "SERVICE_TOKEN") -> str:
    value = os.environ.get(name, "").strip()
    if not value:
        raise RuntimeError(f"required environment variable {name} is missing")
    return value


if __name__ == "__main__":
    token = require_token()
    print(f"token configured: {bool(token)}")
```

示例只确认变量存在，不会验证令牌是否有效，也不会把令牌值输出到终端。生产环境应通过安全的凭据管理机制注入秘密，并避免把秘密复制到命令参数。

## 轮换演练

1. 创建新令牌并赋予最小权限。
2. 将应用切换到新令牌并验证关键操作。
3. 撤销旧令牌。
4. 检查旧令牌是否仍能访问资源。
5. 记录轮换时间、负责人和结果，但不记录令牌本身。

令牌轮换不能只停留在文档里。要在不影响生产服务的条件下定期演练，确认权限边界、失效机制和恢复流程都符合预期。
