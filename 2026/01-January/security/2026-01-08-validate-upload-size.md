---
title: "文件上传必须限制大小和类型"
date: 2026-01-08
category: security
tags: [security, upload, validation, web]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 文件上传必须限制大小和类型

文件上传入口经常被当成普通表单处理，但文件可能占用大量磁盘和内存，也可能带有与扩展名不一致的内容。仅检查文件名后缀，无法证明文件真实类型；只在上传完成后检查大小，也可能已经消耗大量资源。

## 先做分层限制

1. 在反向代理或 Web 框架入口设置请求体大小上限。
2. 应用层再次检查实际读取的字节数。
3. 将文件保存到非公开目录，并使用服务端生成的文件名。
4. 按业务需求验证内容类型，必要时使用专门解析器。
5. 限制上传频率，并设置存储配额和清理策略。

下面的函数演示应用层的字节数限制：

```python
def read_limited(stream, max_bytes: int) -> bytes:
    if max_bytes <= 0:
        raise ValueError("max_bytes must be positive")
    data = stream.read(max_bytes + 1)
    if len(data) > max_bytes:
        raise ValueError("file exceeds size limit")
    return data


if __name__ == "__main__":
    from io import BytesIO

    assert read_limited(BytesIO(b"abc"), 3) == b"abc"
    try:
        read_limited(BytesIO(b"abcd"), 3)
    except ValueError:
        print("oversize file blocked")
    else:
        raise AssertionError("oversize file was accepted")
```

此函数适合演示小型流的限制逻辑，但它把允许范围内的内容一次性读入内存，不适合无限大的文件。生产实现应分块读取并在超限时停止，同时由上游组件限制整个请求体。

## 类型检查的边界

扩展名、浏览器提交的 MIME 类型和文件魔数都只能提供不同程度的线索。高风险格式应在隔离环境中解析；不要在 Web 进程中执行上传文件。若业务允许，优先采用有限格式白名单。

## 验收测试

至少覆盖：大小刚好等于上限、超出一个字节、空文件、伪造扩展名、读取中断和存储失败。拒绝上传时不要留下半成品文件，错误日志也不要记录文件的敏感内容。
