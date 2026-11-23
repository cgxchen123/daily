---
title: "发布 Python wheel 前，先核对包内文件与 RECORD"
date: 2026-10-23
category: open-source
tags: [python, wheel, packaging, supply-chain, integrity]
difficulty: intermediate
reading_time: 12
python_version: "3.10+"
dependencies: ["Python standard library"]
series: "Daily-Tech-Share"
---

# 发布 Python wheel 前，先核对包内文件与 RECORD

> 补档说明：本文归档日期为 2026-10-23；本次实际提交时间以 Git 提交历史为准，不代表文章在归档日期当天已经发布。

## 真实痛点

Python 包从源码构建出 wheel 后，常见做法是直接把文件上传到包仓库。问题在于，“构建命令没有报错”并不能证明最终归档完整：打包配置可能漏掉资源文件，构建目录可能残留旧文件，传输或后处理也可能改变归档内容。开发者在本地运行源码没有问题，用户安装发布包时却可能遇到模块缺失、资源找不到或安装校验失败。

wheel 是 ZIP 格式的二进制分发归档。它的 <code>.dist-info</code> 目录至少包含 <code>METADATA</code>、<code>WHEEL</code> 和 <code>RECORD</code>。其中 <code>RECORD</code> 记录归档中几乎所有文件的路径、哈希和大小；除 <code>RECORD</code> 自身外，文件应有哈希，算法至少达到 SHA-256。安装器会依据记录校验文件内容。因此，在发布前独立读取 wheel 并核对 <code>RECORD</code>，可以提前发现文件缺失、哈希不一致和归档结构异常。

这个检查解决的是“归档内容是否与自身记录一致”，不是“这个包是否来自可信作者”。攻击者完全可以同时替换文件和重算 RECORD；哈希一致不等于来源可信，也不等于代码安全。

## 核心原理

检查器做六件事：

1. 把 wheel 当 ZIP 只读打开，不安装、不执行其中代码，也不把成员解压到文件系统。
2. 限制归档条目数量、解压后声明总大小和 RECORD 文件大小，降低意外超大归档带来的资源风险。
3. 拒绝绝对路径、盘符路径、反斜杠、空路径段、当前目录和上级目录路径；同时拒绝重复 ZIP 成员名，避免不同工具对同一归档产生不同解释。
4. 定位唯一的 <code>.dist-info</code> 目录，确认必需的三个元数据文件存在。
5. 解析 RECORD 的三列 CSV，验证每个普通文件都被记录、每一行路径唯一、文件大小一致、哈希一致；RECORD 自身的哈希和大小必须留空。
6. 允许旧式 <code>RECORD.jws</code> 和 <code>RECORD.p7s</code> 作为未列入 RECORD 的可选文件，但不把它们误当成普通数据文件。

官方规范说明了 <code>.dist-info</code> 的必需文件、RECORD 的哈希要求，以及旧式签名文件的处理方式：<https://packaging.python.org/en/latest/specifications/binary-distribution-format/>。规范已将 <code>RECORD.jws</code> 和 <code>RECORD.p7s</code> 标记为弃用；本工具不会验证数字签名，也不会把“哈希匹配”包装成签名验证。

## 完整可运行代码

保存为 <code>wheel_record_audit.py</code>。只依赖 Python 3.10+ 标准库，不需要安装第三方包。

~~~python
from __future__ import annotations

import argparse
import base64
import csv
import hashlib
import io
import sys
import zipfile
from pathlib import Path, PurePosixPath

OPTIONAL_SIGNATURES = {"RECORD.jws", "RECORD.p7s"}
BLOCK_SIZE = 1024 * 1024
MAX_ENTRIES = 10_000
MAX_UNCOMPRESSED_BYTES = 512 * 1024 * 1024
MAX_RECORD_BYTES = 8 * 1024 * 1024


class WheelAuditError(Exception):
    """Raised when a wheel archive does not satisfy the checks in this tool."""


def safe_archive_path(name: str) -> None:
    if not name or "\\" in name or "\x00" in name or name.startswith("/"):
        raise WheelAuditError(f"不安全的归档路径: {name!r}")

    # ZIP directory entries end with '/', so remove only that final separator.
    normalized = name[:-1] if name.endswith("/") else name
    parts = normalized.split("/")
    if not normalized or any(part in {"", ".", ".."} for part in parts):
        raise WheelAuditError(f"不安全的归档路径: {name!r}")
    if len(parts[0]) >= 2 and parts[0][1] == ":":
        raise WheelAuditError(f"不安全的归档路径: {name!r}")


def hash_member(
    archive: zipfile.ZipFile,
    info: zipfile.ZipInfo,
    algorithm: str,
) -> tuple[str, int]:
    digest = hashlib.new(algorithm)
    total = 0
    with archive.open(info, "r") as stream:
        while True:
            chunk = stream.read(BLOCK_SIZE)
            if not chunk:
                break
            total += len(chunk)
            if total > info.file_size or total > MAX_UNCOMPRESSED_BYTES:
                raise WheelAuditError(
                    f"实际解压大小超过声明值或安全上限: {info.filename}"
                )
            digest.update(chunk)

    if total != info.file_size:
        raise WheelAuditError(f"归档声明大小与实际大小不一致: {info.filename}")
    encoded = base64.urlsafe_b64encode(digest.digest()).decode("ascii").rstrip("=")
    return encoded, total


def read_small_member(archive: zipfile.ZipFile, info: zipfile.ZipInfo) -> bytes:
    if info.file_size > MAX_RECORD_BYTES:
        raise WheelAuditError(f"文件过大，拒绝读取为元数据: {info.filename}")

    data = bytearray()
    with archive.open(info, "r") as stream:
        while True:
            chunk = stream.read(min(BLOCK_SIZE, MAX_RECORD_BYTES + 1 - len(data)))
            if not chunk:
                break
            data.extend(chunk)
            if len(data) > MAX_RECORD_BYTES or len(data) > info.file_size:
                raise WheelAuditError(
                    f"文件超过元数据大小上限或声明大小: {info.filename}"
                )

    if len(data) != info.file_size:
        raise WheelAuditError(f"归档声明大小与实际大小不一致: {info.filename}")
    return bytes(data)


def validate_wheel(path: Path) -> dict[str, int | str]:
    if path.suffix.lower() != ".whl":
        raise WheelAuditError("输入文件扩展名必须是 .whl")
    if not path.is_file():
        raise WheelAuditError(f"文件不存在或不是普通文件: {path}")
    if not zipfile.is_zipfile(path):
        raise WheelAuditError("输入不是有效的 ZIP/wheel 文件")

    try:
        with zipfile.ZipFile(path, "r") as archive:
            infos = archive.infolist()
            if not infos:
                raise WheelAuditError("wheel 归档为空")
            if len(infos) > MAX_ENTRIES:
                raise WheelAuditError(f"归档条目超过上限 {MAX_ENTRIES}")

            total_uncompressed = sum(info.file_size for info in infos)
            if total_uncompressed > MAX_UNCOMPRESSED_BYTES:
                raise WheelAuditError(
                    f"解压后声明大小超过上限 {MAX_UNCOMPRESSED_BYTES} 字节"
                )

            by_name: dict[str, zipfile.ZipInfo] = {}
            info_dirs: set[str] = set()
            for info in infos:
                safe_archive_path(info.filename)
                if info.filename in by_name:
                    raise WheelAuditError(f"归档中存在重复路径: {info.filename}")
                by_name[info.filename] = info

                if not info.is_dir():
                    for part in PurePosixPath(info.filename).parts:
                        if part.endswith(".dist-info"):
                            info_dirs.add(part)

            if len(info_dirs) != 1:
                raise WheelAuditError(
                    f"应当且只能识别一个 .dist-info 目录，实际为 {len(info_dirs)} 个"
                )

            dist_info_name = next(iter(info_dirs))
            # The normal wheel layout has the .dist-info directory at archive root.
            record_path = f"{dist_info_name}/RECORD"
            required_paths = {
                f"{dist_info_name}/METADATA",
                f"{dist_info_name}/WHEEL",
                record_path,
            }
            missing = required_paths - by_name.keys()
            if missing:
                raise WheelAuditError(f"缺少必需的 .dist-info 文件: {sorted(missing)}")
            if any(by_name[item].is_dir() for item in required_paths):
                raise WheelAuditError(".dist-info 必需路径必须是文件")

            record_bytes = read_small_member(archive, by_name[record_path])
            try:
                record_text = record_bytes.decode("utf-8")
            except UnicodeDecodeError as exc:
                raise WheelAuditError("RECORD 不是有效 UTF-8") from exc

            rows: dict[str, tuple[str, str]] = {}
            try:
                reader = csv.reader(io.StringIO(record_text, newline=""))
                for line_number, row in enumerate(reader, start=1):
                    if len(row) != 3:
                        raise WheelAuditError(
                            f"RECORD 第 {line_number} 行必须恰好包含 3 列"
                        )

                    member_name, hash_field, size_field = row
                    safe_archive_path(member_name)
                    if member_name in rows:
                        raise WheelAuditError(f"RECORD 存在重复路径: {member_name}")
                    rows[member_name] = (hash_field, size_field)
            except csv.Error as exc:
                raise WheelAuditError(f"RECORD CSV 格式错误: {exc}") from exc

            if record_path not in rows:
                raise WheelAuditError("RECORD 必须包含自身条目")

            record_hash, record_size = rows[record_path]
            if record_hash or record_size:
                raise WheelAuditError("RECORD 自身的 hash 和 size 必须留空")

            archive_files = {
                name for name, info in by_name.items() if not info.is_dir()
            }
            allowed_unrecorded = {
                f"{dist_info_name}/{signature}" for signature in OPTIONAL_SIGNATURES
            }
            expected_recorded = (
                archive_files - {record_path} - (archive_files & allowed_unrecorded)
            )
            actual_recorded = set(rows) - {record_path}

            missing_rows = expected_recorded - actual_recorded
            extra_rows = actual_recorded - archive_files
            if missing_rows:
                raise WheelAuditError(
                    f"以下归档文件未出现在 RECORD: {sorted(missing_rows)}"
                )
            if extra_rows:
                raise WheelAuditError(
                    f"RECORD 引用了归档中不存在的文件: {sorted(extra_rows)}"
                )

            checked = 0
            for member_name in sorted(expected_recorded):
                hash_field, size_field = rows[member_name]
                if not hash_field or not size_field:
                    raise WheelAuditError(f"文件缺少 hash 或 size: {member_name}")
                if "=" not in hash_field:
                    raise WheelAuditError(f"hash 字段格式错误: {member_name}")

                algorithm, expected_digest = hash_field.split("=", 1)
                if algorithm.lower() in {"md5", "sha1"}:
                    raise WheelAuditError(
                        f"不允许使用 {algorithm}: {member_name}"
                    )
                try:
                    algorithm_probe = hashlib.new(algorithm)
                except (ValueError, TypeError) as exc:
                    raise WheelAuditError(
                        f"不支持的 hash 算法 {algorithm}: {member_name}"
                    ) from exc

                if algorithm_probe.digest_size < 32:
                    raise WheelAuditError(
                        "hash 算法强度低于 SHA-256 或不支持固定摘要长度: "
                        f"{algorithm}"
                    )

                try:
                    expected_size = int(size_field)
                except ValueError as exc:
                    raise WheelAuditError(
                        f"size 不是整数: {member_name}"
                    ) from exc
                if expected_size < 0:
                    raise WheelAuditError(f"size 不能为负数: {member_name}")

                actual_digest, actual_size = hash_member(
                    archive, by_name[member_name], algorithm
                )
                if actual_size != expected_size:
                    raise WheelAuditError(
                        f"文件大小不匹配: {member_name}，"
                        f"RECORD={expected_size}，实际={actual_size}"
                    )
                if actual_digest != expected_digest:
                    raise WheelAuditError(f"文件 hash 不匹配: {member_name}")
                checked += 1

            for signature_path in archive_files & allowed_unrecorded:
                # The wheel specification allows these legacy signature files to be omitted from RECORD.
                if signature_path in rows:
                    raise WheelAuditError(
                        f"旧式签名文件不应写入 RECORD: {signature_path}"
                    )

            return {
                "archive_entries": len(infos),
                "checked_files": checked,
                "uncompressed_bytes": total_uncompressed,
                "dist_info": dist_info_name,
            }
    except (zipfile.BadZipFile, RuntimeError, OSError) as exc:
        raise WheelAuditError(f"读取 wheel 失败: {exc}") from exc


def main() -> int:
    parser = argparse.ArgumentParser(
        description="只读检查 wheel 文件与 .dist-info/RECORD 的完整性，不安装或解压文件。"
    )
    parser.add_argument("wheel", type=Path, help="要检查的 .whl 文件")
    args = parser.parse_args()

    try:
        result = validate_wheel(args.wheel)
    except WheelAuditError as exc:
        print(f"检查失败: {exc}", file=sys.stderr)
        return 1

    print("检查通过")
    print(f".dist-info: {result['dist_info']}")
    print(f"归档条目: {result['archive_entries']}")
    print(f"已核对文件: {result['checked_files']}")
    print(f"解压后声明大小: {result['uncompressed_bytes']} bytes")
    print(
        "注意：哈希一致只证明归档内容与 RECORD 一致，"
        "不证明来源可信或代码安全。"
    )
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
~~~

## 运行与验证

先生成待发布 wheel，例如在项目根目录执行：

~~~bash
python -m pip wheel --no-deps --no-build-isolation --wheel-dir dist .
~~~

这里的构建命令假设项目构建后端及其构建依赖已经准备好；若项目需要隔离构建环境，应按项目自己的构建说明准备依赖，不要为了让示例通过而忽略构建失败。

检查生成的文件：

~~~bash
python wheel_record_audit.py dist/your_package-1.0.0-py3-none-any.whl
~~~

成功时会输出检查通过、识别出的 <code>.dist-info</code> 目录、归档条目数、完成哈希核对的文件数和解压后声明大小。遇到 RECORD 缺行、文件大小或哈希不匹配、路径不安全、重复成员或缺少必要元数据时，程序会返回非零退出码，并给出失败原因。

本次代码在 Python 3.13.5 环境下完成了真实执行测试：构造了一个 RECORD 正确的合成 wheel，并验证通过；随后分别测试被修改的文件、RECORD 缺少文件条目、路径穿越、重复 ZIP 成员、MD5 哈希和错误文件大小，以上异常样例均被拒绝。这个结果只说明这些已执行的场景通过，不代表已经覆盖所有构建后端、操作系统和第三方 wheel。

## 工程思考

将校验放在发布前，而不是等用户安装时报错，能把问题定位在构建产物阶段。更重要的是，发布流水线应保留原始 wheel、检查命令、工具版本、检查结果和最终上传文件的 SHA-256。检查对象必须是将要上传的那个文件；若检查之后又重新打包或修改文件，之前的结果就不能证明新文件通过校验。

资源限制也需要明确。示例最多接受 10,000 个归档条目、512 MiB 的声明解压总大小，以及 8 MiB 的 RECORD 文件；这些是脚本的保护阈值，不是 wheel 规范规定的上限。遇到体积更大的合法包，可以根据项目发布策略审慎调整，但不应删除限制后不加评估地读取不可信归档。

## 升级方向与局限

可以把检查器接入发布前任务，并在构建结束后生成独立的 SHA-256 清单；上传后再下载仓库中的产物，对比上传前后哈希。还可以结合可信发布身份、来源证明、签名验证和依赖漏洞扫描，形成相互补充的供应链证据。

当前脚本不安装包、不执行代码、不验证发布者身份、不验证数字签名，也不判断许可证是否兼容；它只核对 wheel 内部文件与 RECORD 的一致性。若需要做正式发布决策，仍应结合源码审查、构建来源、测试结果和可信发布流程，不能仅凭“检查通过”就认定软件安全。
