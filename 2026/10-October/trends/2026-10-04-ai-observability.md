---
title: "从 Demo 到可维护系统：给 AI 应用加一层可观测性"
date: 2026-10-04
category: trends
tags: [ai-application, observability, logging, python]
difficulty: intermediate
reading_time: 10 min
python_version: "3.10+"
dependencies: []
series: null
---

# 从 Demo 到可维护系统：给 AI 应用加一层可观测性

很多 AI 项目在演示阶段看起来没有问题：输入一段文本，模型返回结果，页面把结果展示出来。真正开始给别人使用后，问题会迅速出现：同一个输入为什么这次变慢了？某个请求为什么失败？模型输出变差，是提示词改了、上下文变长了，还是后处理代码出了问题？

如果系统只打印一句“调用成功”，这些问题几乎无法定位。可观测性不是把日志写得越多越好，而是让每一次请求都留下足够的信息，能够回答三个问题：发生了什么、为什么发生、下一步应该查哪里。

本文使用 Python 3.10+ 写一个可运行的最小示例：模拟 AI 请求，为每次调用记录请求 ID、耗时、输入长度、输出长度、结果状态，并在程序结束时生成统计报告。示例不连接真实模型，但结构可以迁移到云端模型、本地模型或普通 HTTP 服务。

## 一、先定义需要观察的对象

一个可维护的 AI 请求，至少应该有：

- request_id：串起一次请求的唯一标识；
- model：使用的模型或后端名称；
- input_chars：输入字符数，用于观察上下文膨胀；
- output_chars：输出字符数，用于发现空响应或异常截断；
- latency_ms：端到端耗时；
- status：成功、失败或空输出；
- error_type：失败类型，而不是一条模糊错误消息。

日志要记录定位问题所需的事实，但不要把用户原文、密钥、Cookie 等敏感数据直接写进去。实际系统通常记录长度、哈希或脱敏后的摘要，而不是完整输入。

## 二、完整可运行示例

下面只使用标准库。程序会模拟 8 次请求，并故意加入超时与空输出分支。

\`\`\`python
from __future__ import annotations

import hashlib
import json
import random
import time
import uuid
from dataclasses import asdict, dataclass
from typing import Optional


@dataclass
class RequestMetric:
    request_id: str
    model: str
    input_chars: int
    output_chars: int
    latency_ms: float
    status: str
    error_type: Optional[str]
    input_fingerprint: str


class MockAiClient:
    def __init__(self, model_name: str, random_seed: int = 7) -> None:
        self.model_name = model_name
        self.random_generator = random.Random(random_seed)

    def generate(self, prompt: str) -> str:
        time.sleep(self.random_generator.uniform(0.05, 0.25))
        outcome = self.random_generator.random()

        if outcome < 0.12:
            raise TimeoutError("model request timed out")
        if outcome < 0.24:
            return ""
        return f"已处理：{prompt[:24]}"


class ObservableAiService:
    def __init__(self, client: MockAiClient) -> None:
        self.client = client
        self.metrics: list[RequestMetric] = []

    @staticmethod
    def _fingerprint(text: str) -> str:
        return hashlib.sha256(text.encode("utf-8")).hexdigest()[:12]

    def ask(self, prompt: str) -> str:
        request_id = uuid.uuid4().hex[:12]
        started_at = time.perf_counter()
        output_text = ""
        error_type: Optional[str] = None
        status = "success"

        try:
            output_text = self.client.generate(prompt)
            if not output_text:
                status = "empty_output"
                error_type = "EmptyOutput"
        except TimeoutError:
            status = "failed"
            error_type = "TimeoutError"
        except Exception as unexpected_error:
            status = "failed"
            error_type = type(unexpected_error).__name__

        latency_ms = round((time.perf_counter() - started_at) * 1000, 2)
        metric = RequestMetric(
            request_id=request_id,
            model=self.client.model_name,
            input_chars=len(prompt),
            output_chars=len(output_text),
            latency_ms=latency_ms,
            status=status,
            error_type=error_type,
            input_fingerprint=self._fingerprint(prompt),
        )
        self.metrics.append(metric)
        print(json.dumps(asdict(metric), ensure_ascii=False))
        return output_text

    def report(self) -> dict[str, object]:
        total_requests = len(self.metrics)
        successful_requests = sum(m.status == "success" for m in self.metrics)
        failed_requests = sum(m.status == "failed" for m in self.metrics)
        latencies = [m.latency_ms for m in self.metrics]

        return {
            "total_requests": total_requests,
            "success_rate": round(successful_requests / total_requests, 3)
            if total_requests else 0.0,
            "failure_rate": round(failed_requests / total_requests, 3)
            if total_requests else 0.0,
            "empty_output_count": sum(
                m.status == "empty_output" for m in self.metrics
            ),
            "average_latency_ms": round(sum(latencies) / len(latencies), 2)
            if latencies else 0.0,
            "max_latency_ms": max(latencies) if latencies else 0.0,
        }


def main() -> None:
    service = ObservableAiService(MockAiClient(model_name="demo-model"))
    prompts = [
        "解释什么是幂等性，并给一个接口设计例子。",
        "把需求拆成三个可执行任务。",
        "写一个函数统计日志中的错误类型。",
        "为什么超时不能无限重试？",
        "给出一个缓存键设计。",
        "解释输入长度为什么会影响延迟。",
        "如何为模型调用设置失败边界？",
        "把技术说明改写成检查清单。",
    ]

    for prompt in prompts:
        service.ask(prompt)

    print("\\n汇总报告：")
    print(json.dumps(service.report(), ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
\`\`\`

保存为 \`observable_ai_service.py\`，执行：

\`\`\`bash
python observable_ai_service.py
\`\`\`

每次请求会输出一行 JSON，最后输出汇总报告。固定随机种子让状态更容易复现，但耗时仍受本机调度影响，因此不要把该输出当成性能基准。

## 三、为什么记录指纹，而不是直接记录输入

完整输入在调试时很方便，但生产系统往往不能这样做。用户输入可能包含手机号、订单信息、内部文档甚至凭据。这里使用 SHA-256 生成短指纹有两个作用：同一个输入可以被关联；日志里不出现原始内容，降低敏感信息泄露风险。

指纹不是加密，也不是匿名化的万能方案。短输入仍可能被猜测，生产环境需要结合访问控制、日志保留周期、脱敏策略和加密存储一起使用。

## 四、工程化上还缺什么

### 1. 日志和指标要分开

日志适合记录单次请求的详细事实，指标适合回答总体趋势，例如每分钟请求量、P50/P95/P99 延迟、各类错误比例、不同模型的成本和成功率。数据量变大后，不能所有问题都靠搜索日志解决。

### 2. 重试必须有边界

超时不等于可以无限重试。至少要限制最大重试次数、单次请求总超时时间、哪些错误允许重试，以及重试是否会造成重复扣费或重复写入。对有副作用的操作，必须先设计幂等键。

### 3. 关联 ID 要贯穿调用链

如果前面还有网关、任务队列和数据库，实际项目通常需要同时保留 trace_id、span_id 和 request_id。这样才能从网页请求追到模型调用，再追到后处理和持久化。

### 4. 结构化日志优于拼接字符串

\`print("调用成功，耗时 123ms")\` 对人看起来简单，但机器很难稳定解析。结构化 JSON 日志更适合被日志系统消费，也方便后续聚合统计。

## 五、如何迁移到真实模型调用

替换 \`MockAiClient.generate()\` 即可，外围的观测逻辑不要动。建议把模型调用、观测、业务校验、日志落盘分层。以后无论换模型供应商还是 SDK，指标字段仍然保持一致，历史数据也能继续比较。

## 六、升级方向

1. 为 \`report()\` 增加 P95 延迟计算；
2. 将 JSON 日志写入按天滚动的文件；
3. 增加 token 数和费用字段；
4. 加入超时、重试和熔断策略；
5. 用 OpenTelemetry 接入分布式追踪；
6. 为不同模型建立质量、延迟、成本三维对比。

这篇文章的重点不是“打印更多日志”，而是把一次不可解释的模型调用，变成一条可以被追踪、统计和改进的工程记录。AI 应用从 Demo 走向长期运行时，真正需要补上的往往不是更多提示词，而是这些基础设施。