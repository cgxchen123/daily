---
title: "引入开源项目之前，先核对许可证和使用方式"
date: 2026-01-15
category: open-source
tags: [open-source, license, compliance]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 引入开源项目之前，先核对许可证和使用方式

公开仓库不等于可以不受限制地复制、修改或分发。许可证可能要求保留版权声明、附带许可证文本、公开衍生作品源码，或对专利和商标作出特定约定。只看 README 中的“open source”描述不够。

## 依赖审查的最低步骤

1. 找到仓库根目录的 LICENSE 文件，并确认它适用于当前版本。
2. 核对包管理器中安装的版本与许可证来源是否一致。
3. 检查依赖树，区分直接依赖和传递依赖。
4. 记录版权声明、通知文件和分发义务。
5. 对含有例外条款、双许可证或许可证缺失的项目交由合规负责人确认。

## 用清单记录审查结果

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class DependencyReview:
    name: str
    version: str
    license_id: str
    source_url: str
    reviewed: bool


def ready_for_release(item: DependencyReview) -> bool:
    return bool(
        item.name.strip()
        and item.version.strip()
        and item.license_id.strip()
        and item.source_url.startswith("https://")
        and item.reviewed
    )


if __name__ == "__main__":
    sample = DependencyReview(
        "example-lib", "1.2.3", "MIT",
        "https://example.org/project", True
    )
    assert ready_for_release(sample)
    print("review record complete")
```

示例只检查审查记录是否填写完整，不会自动判断许可证是否合法，也不能替代法律意见。

## 常见误区

- “代码免费”不代表没有分发条件。
- 许可证标识符可能对应多个版本或例外条款。
- 包仓库的元数据可能缺失或错误，需要回到作者发布的源码版本核对。
- 公司产品是否构成分发、是否涉及专利条款，要结合具体使用场景判断。

把依赖许可证审查纳入发布流程，比在产品上线后才追溯来源更可控。重要的是留存版本、来源、审查人和结论，而不是只保存一个许可证名称。
