---
title: "从一开始就把 Python 项目目录分清楚"
date: 2026-01-01
category: engineering
tags: [python, project-structure, maintainability]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 从一开始就把 Python 项目目录分清楚

一个脚本刚开始只有几十行时，把所有逻辑放进一个文件很方便；当它逐渐加入配置、文件读写和测试后，单文件就会让依赖关系变得难以理解。目录结构不需要一开始就复杂，但至少要区分“可复用逻辑”“程序入口”和“测试”。

## 推荐的最小结构

```text
sample-app/
├── pyproject.toml
├── README.md
├── src/
│   └── sample_app/
│       └── __init__.py
├── tests/
│   └── test_core.py
└── scripts/
```

`src` 布局可以减少测试时意外导入仓库根目录同名文件的风险；测试放在独立目录，能让验证命令成为项目的一部分。不要为了看起来专业就过早拆成十几个包。

## 一个可验证的最小函数

将业务逻辑写成不依赖终端输入的函数，测试就容易重复执行：

```python
def normalize_name(value: str) -> str:
    result = value.strip()
    if not result:
        raise ValueError("name must not be empty")
    return result


if __name__ == "__main__":
    assert normalize_name("  Ada  ") == "Ada"
    try:
        normalize_name("   ")
    except ValueError:
        pass
    else:
        raise AssertionError("empty name should fail")
    print("checks passed")
```

保存为 `core.py` 后运行 `python core.py`。两个断言分别覆盖正常输入和非法输入。项目正式采用包结构后，应将函数移入包，并在测试目录使用测试框架调用。

## 实践建议

- 先按职责分文件，而不是按代码行数机械拆分。
- 配置、业务逻辑和 I/O 尽量分离。
- README 写清安装、运行、测试三条命令。
- 不要把本机路径、令牌或真实用户数据提交进仓库。

小项目最重要的不是目录多漂亮，而是别人能快速找到入口、运行示例并验证行为。
