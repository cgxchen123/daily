---
title: "批处理任务要能安全重跑：从幂等性开始"
date: 2026-01-05
category: automation
tags: [automation, idempotency, python, reliability]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 批处理任务要能安全重跑：从幂等性开始

定时任务可能因为网络超时、进程中断或调度器重试而重复执行。若每次运行都会重复追加数据、重复发通知或覆盖已有成果，偶发故障就会变成数据问题。幂等设计的目标是：同一份输入重复处理，不会产生重复的业务结果。

## 用稳定标识去重

下面的本地示例以事件 ID 作为幂等键，演示重复记录如何被跳过：

```python
def process_events(events: list[dict], processed_ids: set[str]) -> list[dict]:
    new_items = []
    for event in events:
        event_id = event.get("id")
        if not isinstance(event_id, str) or not event_id:
            raise ValueError("every event needs a non-empty id")
        if event_id in processed_ids:
            continue
        new_items.append(event)
        processed_ids.add(event_id)
    return new_items


if __name__ == "__main__":
    seen = {"evt-1"}
    batch = [{"id": "evt-1"}, {"id": "evt-2"}]
    result = process_events(batch, seen)
    assert result == [{"id": "evt-2"}]
    assert process_events(batch, seen) == []
    print("idempotency checks passed")
```

保存并执行后，第二次处理相同批次不会再次输出事件。这个示例只使用内存集合，程序退出后状态就会消失，因此它不是生产环境的持久化实现。

## 生产环境还需要什么

1. 将幂等键保存在数据库，并对该键建立唯一约束。
2. 把“检查是否处理过”和“写入结果”放在同一个事务里。
3. 明确处理失败时的状态：可重试、永久失败或等待人工介入。
4. 对外部 API 使用供应商支持的 idempotency key；没有支持时不要假设网络重试天然安全。
5. 记录输入标识、结果状态和尝试次数，便于审计。

单纯在程序开始时查询一次“是否处理过”，然后在稍后写入，仍可能被两个并发任务同时通过。可靠幂等依赖稳定标识、持久化约束和原子更新，而不仅是一个 if 判断。
