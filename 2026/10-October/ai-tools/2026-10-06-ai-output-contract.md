---
title: "别只给 AI 写提示词：给输出加一个可验证的契约"
date: 2026-10-06
category: ai-tools
tags: [prompt-engineering, structured-output, validation, automation]
difficulty: beginner
reading_time: 9 min
python_version: "3.10+"
dependencies: []
series: null
---

# 别只给 AI 写提示词：给输出加一个可验证的契约

很多 AI 工具第一次做出来很好用，接进自动化流程后却开始变得脆弱。

原因通常不是模型突然“变笨”，而是系统把模型的自然语言输出直接当成了程序输入。比如你要求 AI 返回标题、摘要和标签，第一次可能得到标准 JSON，下一次却多了一段解释；字段名变了；标签从数组变成了逗号分隔的字符串。对人来说都能看懂，对程序来说却可能直接失败。

一个更稳的思路是：**提示词负责约束模型，程序负责验证结果。**

两者不要互相替代。

## 一、先把“想要什么”写成输出契约

与其只写：

> 帮我整理这段内容，返回标题、摘要和标签。

不如明确告诉模型：

- 顶层必须是 JSON 对象；
- 必须包含 `title`、`summary`、`tags`；
- `title` 和 `summary` 必须是字符串；
- `tags` 必须是 1～5 个字符串组成的数组；
- 不要输出 JSON 之外的解释。

可以直接使用这样的提示词：

```text
你是一个内容结构化助手。

任务：
把输入内容整理成结构化元数据。

输出契约：
1. 只能输出一个 JSON 对象，不要 Markdown，不要解释。
2. 必须包含：
   - title：非空字符串
   - summary：非空字符串
   - tags：1～5 个非空字符串组成的数组
3. 不得增加未定义的顶层字段。
4. 无法确定的信息不要编造，使用已有输入能够支持的内容。

输出示例：
{
  "title": "Git 提交前检查",
  "summary": "检查关键文件，降低自动发布出错概率。",
  "tags": ["git", "automation"]
}

输入：
{{content}}
```

这已经比“请返回 JSON”可靠很多，但仍然不能把模型输出直接交给下游。

## 二、为什么还要程序验证

提示词不是类型系统。

即使模型理解了要求，也可能因为上下文、模型版本、生成策略或输入内容发生变化而产生不符合契约的结果。

因此实际链路应该是：

```
用户输入
   ↓
提示词 + 输出契约
   ↓
AI 模型
   ↓
原始输出
   ↓
JSON 解析
   ↓
字段/类型/范围校验
   ↓
通过 → 进入业务流程
失败 → 拒绝 / 重试 / 人工处理
```

关键点在最后三步：**模型输出只是候选结果，不是可信数据。**

## 三、一个不依赖第三方库的验证器

下面用 Python 3.10+ 写一个最小但完整的验证程序。它不调用任何 AI API，输入可以直接来自任意模型或人工生成的文本，因此可以先独立验证下游边界。

```python
from __future__ import annotations

import json
import sys
from typing import Any


REQUIRED = {"title", "summary", "tags"}


def validate_result(raw: str) -> dict[str, Any]:
    try:
        data = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise ValueError(f"输出不是合法 JSON：{exc.msg}") from exc

    if not isinstance(data, dict):
        raise ValueError("顶层必须是 JSON 对象")

    missing = REQUIRED - data.keys()
    if missing:
        raise ValueError(f"缺少字段：{', '.join(sorted(missing))}")

    if not isinstance(data["title"], str) or not data["title"].strip():
        raise ValueError("title 必须是非空字符串")

    if not isinstance(data["summary"], str) or not data["summary"].strip():
        raise ValueError("summary 必须是非空字符串")

    tags = data["tags"]
    if (
        not isinstance(tags, list)
        or not 1 <= len(tags) <= 5
        or not all(isinstance(tag, str) and tag.strip() for tag in tags)
    ):
        raise ValueError("tags 必须是 1~5 个非空字符串")

    return data


def main() -> None:
    raw = sys.stdin.read()

    try:
        result = validate_result(raw)
    except ValueError as exc:
        print(f"INVALID: {exc}")
        raise SystemExit(1)

    print(json.dumps(result, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

保存为 `validate_ai_output.py`，不需要安装任何第三方依赖。

合法输入：

```json
{"title":"Git 提交前检查","summary":"检查关键文件","tags":["git","automation"]}
```

运行：

```bash
printf '%s' '{"title":"Git 提交前检查","summary":"检查关键文件","tags":["git","automation"]}' | python validate_ai_output.py
```

验证通过后，程序会重新格式化并输出 JSON。

如果输入：

```json
{"title":"","summary":"x","tags":[]}
```

程序会拒绝它，而不是把错误数据继续传给下游。

## 四、真正接入 AI 工具时怎么用

不要让模型同时承担“生成”和“最终裁决”。

例如一个自动内容系统可以这样设计：

1. Prompt 要求固定 JSON 结构；
2. 模型返回结果；
3. JSON 解析失败，进入有限次数重试；
4. JSON 能解析但字段不符合规则，拒绝；
5. 校验通过，才写入 Markdown、数据库或 Git；
6. 高风险动作继续进入独立审批层。

这里有一个很重要的边界：

**校验器只能判断格式和你明确写下来的规则，不能证明模型说的事实是真的。**

例如 `tags` 是数组，只能说明格式正确，不能说明标签选择一定合理；`summary` 是字符串，也不能说明摘要没有事实错误。

所以结构校验、事实核验、权限控制应该是三层不同的机制。

## 五、工程上最容易踩的坑

### 1. 只校验 JSON，不校验字段

`{"title": 123}` 是合法 JSON，但不是合法业务数据。

### 2. 校验存在，不校验范围

标签是数组不代表它一定应该允许 500 个标签。

### 3. 失败后无限重试

如果提示词本身有问题，无限重试只是在重复消耗资源。实际系统应该设置明确的重试上限。

### 4. 把“解析成功”当成“事实正确”

这是最危险的一层误判。格式正确和内容正确是两件事。

## 六、升级方向

小项目可以继续保持这种手写校验方式；规则变复杂后，可以引入 JSON Schema 或项目已有的数据校验框架，把字段类型、枚举值、长度范围等规则集中管理。

再往后，可以把失败样本保存为回归测试集：以后修改 Prompt、切换模型或调整工作流时，自动重新验证这些历史样本。

这样 Prompt Engineering 才真正从“写一句更好的提示词”，变成了可以测试、可以回归、可以维护的工程工作。

## 结论

AI 输出进入程序之前，最好经过一道明确的边界。

提示词负责告诉模型“应该长什么样”，验证器负责确认“实际是不是这样”，事实核验和权限系统再负责回答“内容是否可信、动作是否允许”。

当 AI 从聊天工具变成自动化系统的一部分时，这个边界比继续堆提示词形容词更重要。
