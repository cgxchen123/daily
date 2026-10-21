---
title: "健康检查要区分进程存活与服务就绪"
date: 2026-01-09
category: engineering
tags: [observability, health-check, reliability, deployment]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 健康检查要区分进程存活与服务就绪

容器或进程管理器通常需要健康检查，但“进程还活着”和“服务现在能接收请求”不是同一个问题。如果检查接口只返回固定的 200，依赖服务已经失效时，调度系统仍可能不断把请求送进故障实例。

## 两种检查解决不同问题

- **存活检查（liveness）**：判断进程是否陷入无法恢复的状态。失败可能触发重启。
- **就绪检查（readiness）**：判断实例是否已经完成启动、配置加载，并能安全处理请求。失败时通常应暂时停止向它分发流量。

不要把所有下游依赖都无条件塞进存活检查。如果数据库短暂故障导致每个实例都被重启，可能扩大故障范围。

## 用简单状态模型表达就绪条件

```python
def is_ready(startup_complete: bool, config_valid: bool) -> bool:
    return startup_complete and config_valid


if __name__ == "__main__":
    assert not is_ready(False, True)
    assert not is_ready(True, False)
    assert is_ready(True, True)
    print("readiness checks passed")
```

这个函数只是业务条件的示例，不是完整的 HTTP 健康检查端点。真实服务还应考虑超时、状态码、依赖检查范围和启动阶段。

## 设计检查接口时

1. 让响应快速、稳定，不返回环境变量或内部凭据。
2. 设定短超时，避免健康检查本身占满线程。
3. 为启动预热设置合理的延迟或启动探针。
4. 记录失败原因和持续时间，但不要让单次短暂波动触发连锁重启。
5. 将“检查失败后的动作”与检查本身分开设计。

健康检查是调度系统的控制信号，不是装饰性接口。检查条件应该与平台在失败后采取的动作相匹配。
