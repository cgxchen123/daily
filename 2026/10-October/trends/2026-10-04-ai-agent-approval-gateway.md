---
title: "AI Agent 开始替你做事后，先给它加一扇可审批的门"
date: 2026-10-04
category: trends
tags: [ai-agent, tool-calling, approval, security, python]
difficulty: intermediate
reading_time: 10 min
python_version: "3.10+"
dependencies: []
series: null
---

# AI Agent 开始替你做事后，先给它加一扇可审批的门

2026 年 9 月 29 日的 OpenAI DevDay 2026 公布了 Dots，官方把它描述为能够持续承担职责、连接应用并开展工作的 Agent；官方产品页同时把访问权限、操作审核与批准列为控制点之一。

这里真正值得开发者关注的不是“模型会不会聊天”，而是：

**当模型开始调用工具、修改数据、发送消息时，谁决定它能不能真的执行？**

如果答案只有“让模型自己决定”，权限边界就会和模型输出绑死。更稳妥的做法是把“模型提出动作”和“系统允许动作”拆开。

本文不调用任何真实模型 API，而是用 Python 3.10+ 实现一个本地可运行的工具调用网关：

- 白名单限制工具；
- 参数结构校验；
- 高风险动作进入人工审批；
- 所有决定写入审计记录。

参考：<https://openai.com/index/devday-2026-recap/>
参考：<https://openai.com/index/introducing-dots/>

## 一、先看清架构边界

一个容易出问题的 Agent 流程是：

    用户 -> 模型 -> 直接调用工具 -> 修改数据

更稳妥的是：

    用户
      ↓
    模型提出工具调用
      ↓
    Policy Gateway
      ├─ 工具是否允许
      ├─ 参数是否合法
      ├─ 风险等级
      └─ 是否需要人工审批
      ↓
    Tool Executor
      ↓
    Audit Log

**模型负责提出“做什么”，策略层负责决定“准不准做”。**

## 二、完整可运行示例

下面只使用 Python 标准库。示例包含三个工具：查文档、创建草稿、发送消息。发送消息是高风险外部副作用，因此必须先审批。

### 1. 数据结构和注册表

~~~~python
from __future__ import annotations

import json
import re
import uuid
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from typing import Any, Callable


@dataclass(frozen=True)
class ToolSpec:
    name: str
    risk: str
    description: str
    handler: Callable[[dict[str, Any]], dict[str, Any]]


@dataclass
class AuditEvent:
    event_id: str
    request_id: str
    tool_name: str
    decision: str
    risk: str
    reason: str
    arguments: dict[str, Any]
    created_at: str


@dataclass
class ApprovalRequest:
    approval_id: str
    request_id: str
    tool_name: str
    arguments: dict[str, Any]
    status: str = "pending"


class ToolRegistry:
    def __init__(self) -> None:
        self._tools: dict[str, ToolSpec] = {}

    def register(self, spec: ToolSpec) -> None:
        if spec.name in self._tools:
            raise ValueError(f"duplicate tool: {spec.name}")
        self._tools[spec.name] = spec

    def get(self, name: str) -> ToolSpec:
        try:
            return self._tools[name]
        except KeyError as exc:
            raise ValueError(f"unknown tool: {name}") from exc
~~~~

### 2. 参数校验和策略引擎

~~~~python
class ArgumentValidator:
    EMAIL_PATTERN = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

    def validate(self, tool_name: str, arguments: dict[str, Any]) -> None:
        if not isinstance(arguments, dict):
            raise ValueError("arguments must be an object")

        if tool_name == "search_docs":
            query = arguments.get("query")
            if not isinstance(query, str) or not query.strip():
                raise ValueError("query must be a non-empty string")
            if len(query) > 200:
                raise ValueError("query is too long")
            return

        if tool_name == "create_draft":
            title = arguments.get("title")
            body = arguments.get("body")
            if not isinstance(title, str) or not title.strip():
                raise ValueError("title must be a non-empty string")
            if not isinstance(body, str) or not body.strip():
                raise ValueError("body must be a non-empty string")
            if len(title) > 120:
                raise ValueError("title is too long")
            return

        if tool_name == "send_message":
            recipient = arguments.get("recipient")
            body = arguments.get("body")
            if not isinstance(recipient, str):
                raise ValueError("recipient must be a string")
            if not self.EMAIL_PATTERN.fullmatch(recipient):
                raise ValueError("recipient is not a valid email address")
            if not isinstance(body, str) or not body.strip():
                raise ValueError("body must be a non-empty string")
            if len(body) > 5000:
                raise ValueError("body is too long")
            return

        raise ValueError(f"no validator for tool: {tool_name}")


@dataclass(frozen=True)
class PolicyResult:
    allowed: bool
    requires_approval: bool
    reason: str


class PolicyEngine:
    def evaluate(
        self,
        spec: ToolSpec,
        arguments: dict[str, Any],
    ) -> PolicyResult:
        if spec.risk == "low":
            return PolicyResult(True, False, "low-risk read operation")

        if spec.risk == "medium":
            return PolicyResult(True, False, "medium-risk draft operation")

        if spec.risk == "high":
            return PolicyResult(
                False,
                True,
                "high-risk external side effect requires approval",
            )

        return PolicyResult(
            False,
            False,
            f"unsupported risk level: {spec.risk}",
        )
~~~~

### 3. 执行器、审计和审批网关

~~~~python
class InMemoryStore:
    def __init__(self) -> None:
        self.documents: list[dict[str, Any]] = []
        self.messages: list[dict[str, Any]] = []

    def search(self, query: str) -> dict[str, Any]:
        q = query.lower()
        matched = [
            doc for doc in self.documents if q in doc["body"].lower()
        ]
        return {"count": len(matched), "documents": matched}

    def draft(self, title: str, body: str) -> dict[str, Any]:
        item = {"id": str(uuid.uuid4()), "title": title, "body": body}
        self.documents.append(item)
        return item

    def send_message(self, recipient: str, body: str) -> dict[str, Any]:
        item = {
            "id": str(uuid.uuid4()),
            "recipient": recipient,
            "body": body,
        }
        self.messages.append(item)
        return item


class AuditLogger:
    def __init__(self) -> None:
        self.events: list[AuditEvent] = []

    def write(
        self,
        request_id: str,
        tool_name: str,
        decision: str,
        risk: str,
        reason: str,
        arguments: dict[str, Any],
    ) -> AuditEvent:
        event = AuditEvent(
            event_id=str(uuid.uuid4()),
            request_id=request_id,
            tool_name=tool_name,
            decision=decision,
            risk=risk,
            reason=reason,
            arguments=arguments,
            created_at=datetime.now(timezone.utc).isoformat(),
        )
        self.events.append(event)
        return event


class ToolGateway:
    def __init__(
        self,
        registry: ToolRegistry,
        validator: ArgumentValidator,
        policy: PolicyEngine,
        audit: AuditLogger,
    ) -> None:
        self.registry = registry
        self.validator = validator
        self.policy = policy
        self.audit = audit
        self.approvals: dict[str, ApprovalRequest] = {}

    def request(
        self,
        tool_name: str,
        arguments: dict[str, Any],
    ) -> dict[str, Any]:
        request_id = str(uuid.uuid4())

        try:
            spec = self.registry.get(tool_name)
        except ValueError as exc:
            self.audit.write(
                request_id,
                tool_name,
                "denied",
                "unknown",
                str(exc),
                arguments,
            )
            return {
                "status": "denied",
                "request_id": request_id,
                "reason": str(exc),
            }

        try:
            self.validator.validate(tool_name, arguments)
        except ValueError as exc:
            self.audit.write(
                request_id,
                tool_name,
                "denied",
                spec.risk,
                str(exc),
                arguments,
            )
            return {
                "status": "denied",
                "request_id": request_id,
                "reason": str(exc),
            }

        decision = self.policy.evaluate(spec, arguments)

        if decision.requires_approval:
            approval = ApprovalRequest(
                approval_id=str(uuid.uuid4()),
                request_id=request_id,
                tool_name=tool_name,
                arguments=arguments.copy(),
            )
            self.approvals[approval.approval_id] = approval

            self.audit.write(
                request_id,
                tool_name,
                "pending_approval",
                spec.risk,
                decision.reason,
                arguments,
            )

            return {
                "status": "pending_approval",
                "request_id": request_id,
                "approval_id": approval.approval_id,
                "reason": decision.reason,
            }

        if not decision.allowed:
            self.audit.write(
                request_id,
                tool_name,
                "denied",
                spec.risk,
                decision.reason,
                arguments,
            )
            return {
                "status": "denied",
                "request_id": request_id,
                "reason": decision.reason,
            }

        result = spec.handler(arguments)

        self.audit.write(
            request_id,
            tool_name,
            "executed",
            spec.risk,
            decision.reason,
            arguments,
        )

        return {
            "status": "executed",
            "request_id": request_id,
            "result": result,
        }

    def approve(self, approval_id: str) -> dict[str, Any]:
        approval = self.approvals.get(approval_id)
        if approval is None:
            raise ValueError("approval request not found")

        if approval.status != "pending":
            raise ValueError(
                f"approval request is already {approval.status}"
            )

        spec = self.registry.get(approval.tool_name)
        result = spec.handler(approval.arguments)
        approval.status = "approved"

        self.audit.write(
            approval.request_id,
            approval.tool_name,
            "approved_and_executed",
            spec.risk,
            "approved by human",
            approval.arguments,
        )

        return {
            "status": "executed",
            "request_id": approval.request_id,
            "approval_id": approval.approval_id,
            "result": result,
        }

    def reject(self, approval_id: str, reason: str) -> dict[str, Any]:
        approval = self.approvals.get(approval_id)
        if approval is None:
            raise ValueError("approval request not found")

        if approval.status != "pending":
            raise ValueError(
                f"approval request is already {approval.status}"
            )

        approval.status = "rejected"

        self.audit.write(
            approval.request_id,
            approval.tool_name,
            "rejected",
            "high",
            reason,
            approval.arguments,
        )

        return {
            "status": "rejected",
            "request_id": approval.request_id,
            "approval_id": approval.approval_id,
            "reason": reason,
        }
~~~~

### 4. 运行主程序

~~~~python
def build_gateway() -> tuple[ToolGateway, InMemoryStore]:
    store = InMemoryStore()
    registry = ToolRegistry()

    registry.register(
        ToolSpec(
            name="search_docs",
            risk="low",
            description="search local documents",
            handler=lambda args: store.search(args["query"]),
        )
    )

    registry.register(
        ToolSpec(
            name="create_draft",
            risk="medium",
            description="create a local draft",
            handler=lambda args: store.draft(
                args["title"],
                args["body"],
            ),
        )
    )

    registry.register(
        ToolSpec(
            name="send_message",
            risk="high",
            description="send an external message",
            handler=lambda args: store.send_message(
                args["recipient"],
                args["body"],
            ),
        )
    )

    gateway = ToolGateway(
        registry,
        ArgumentValidator(),
        PolicyEngine(),
        AuditLogger(),
    )
    return gateway, store


def main() -> None:
    gateway, store = build_gateway()

    print("1) 低风险工具：直接执行")
    print(
        json.dumps(
            gateway.request("search_docs", {"query": "agent"}),
            ensure_ascii=False,
            indent=2,
        )
    )

    print("\n2) 中风险工具：创建草稿")
    print(
        json.dumps(
            gateway.request(
                "create_draft",
                {
                    "title": "Agent 审批策略",
                    "body": "高风险动作必须进入人工审批。",
                },
            ),
            ensure_ascii=False,
            indent=2,
        )
    )

    print("\n3) 高风险工具：进入审批")
    pending = gateway.request(
        "send_message",
        {
            "recipient": "demo@example.com",
            "body": "这是一个演示消息。",
        },
    )
    print(json.dumps(pending, ensure_ascii=False, indent=2))

    print("\n4) 人工批准")
    print(
        json.dumps(
            gateway.approve(pending["approval_id"]),
            ensure_ascii=False,
            indent=2,
        )
    )

    print("\n5) 最终状态")
    print(f"草稿数量：{len(store.documents)}")
    print(f"发送消息数量：{len(store.messages)}")


if __name__ == "__main__":
    main()
~~~~

把四段代码依次合并保存为 agent_tool_gateway.py，然后运行：

~~~~bash
python agent_tool_gateway.py
~~~~

程序只使用标准库，不连接真实模型，也不会向真实邮箱发送消息。

## 三、这个网关为什么值得单独存在

模型输出的是“意图”，而真正有副作用的是工具执行。

例如模型可能提出：

~~~~json
{
  "tool": "send_message",
  "arguments": {
    "recipient": "demo@example.com",
    "body": "请确认合同已经签署。"
  }
}
~~~~

网关仍然需要回答：

- 工具是不是白名单里的；
- 参数是不是合法；
- 当前动作风险有多高；
- 当前用户有没有权限；
- 是否需要人工确认；
- 最终执行是否留下审计记录。

这样即使以后换模型，系统边界也还在。

## 四、为什么不能只把规则写进 Prompt

“发送前必须征得同意”可以写在系统提示词里，但它不应该是最后一道安全边界。

提示词属于模型输入；权限和审批属于程序控制。

对于发送消息、修改数据库、删除文件、发起支付、执行远程命令等外部副作用，最终执行前仍应由程序策略重新判断。

可以把它理解成：

    Prompt       = 操作规章
    Policy       = 门禁
    Audit Log    = 记录

规章不能代替门禁。

## 五、生产环境至少还要补四件事

### 1. 用户级权限

真实系统要从“工具级权限”继续细化到：

    用户 -> 组织 -> 角色 -> 工具 -> 资源

同一个工具，对不同用户可能有完全不同的可操作范围。

### 2. 更严格的参数策略

类型正确不等于业务合法。

真实策略至少应该支持：

- 类型；
- 长度；
- 数值范围；
- 枚举；
- 资源归属；
- 用户权限。

### 3. 审批后的参数必须不可偷偷变化

用户批准的是 A，就只能执行 A。

因此审批记录里要绑定请求参数、请求 ID 和审批 ID；执行阶段不要重新从模型结果取一份参数。

### 4. 外部调用要有超时、重试和幂等

数据库写入、下单、发送消息等动作，至少应该考虑：

    timeout
    retry limit
    idempotency key
    request_id
    audit_id

否则一次网络抖动，就可能把一个动作执行两遍。

## 六、从 Demo 继续演进

这个程序下一步可以按固定顺序升级：

1. 把参数校验换成 JSON Schema；
2. 给工具增加资源范围；
3. 加审批页面；
4. 增加 trace_id / parent_id；
5. 把审计事件写入 SQLite 或日志系统；
6. 给外部 API 增加超时、重试和幂等；
7. 最后再接真实模型的 tool calling。

这样测试可以先围绕策略层完成，不必把每个安全测试都建立在某一次模型输出上。

## 七、结论

当 Agent 从“回答问题”变成“持续执行任务”，系统最需要增加的不是更长的 Prompt，而是一层独立于模型之外的控制面：

**权限、策略、审批、执行、审计。**

模型可以提出“做什么”，但系统应该明确：

**什么情况下允许真的做。**

## 参考资料

- OpenAI DevDay 2026 Recap：<https://openai.com/index/devday-2026-recap/>
- OpenAI Introducing Dots：<https://openai.com/index/introducing-dots/>
