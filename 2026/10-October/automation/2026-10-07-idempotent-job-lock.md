--- 
title: "定时任务最怕重复执行：用 SQLite 做一个可恢复的幂等锁"
date: 2026-10-07
category: automation
tags: [automation, idempotency, sqlite, python]
difficulty: intermediate
reading_time: 11 min
python_version: "3.10+"
dependencies: []
series: null
---

# 定时任务最怕重复执行：用 SQLite 做一个可恢复的幂等锁

很多自动化任务真正难处理的不是“怎么执行”，而是**同一件事被执行两次怎么办**。

例如一个任务本来应该每天处理一次：

\`\`\`text
定时器触发
  ↓
读取数据
  ↓
生成结果
  ↓
发布
\`\`\`

如果调度器重试、进程超时后又启动一次，或者两个实例恰好同时运行，就可能变成：

\`\`\`text
实例 A ──→ 发布一次
实例 B ──→ 又发布一次
\`\`\`

如果任务只是打印日志，重复一次问题不大；但如果任务会发邮件、创建订单、提交 Git、写数据库或者调用有副作用的 API，重复执行就可能造成实际损失。

这篇不做复杂的分布式锁。我们用 Python 标准库里的 SQLite，做一个**本地单机任务的幂等执行门**：

- 同一个任务键在指定时间窗口内只允许成功领取一次；
- 多个进程同时竞争时，由数据库事务决定谁成功；
- 任务执行失败可以释放本次占用；
- 历史记录可以查询；
- 不需要安装第三方包；
- 明确说明它不能替代跨机器分布式锁。

## 一、先区分“防重复”和“幂等”

这两个概念很容易混在一起。

**防重复执行**关注的是：

> 两个实例不要同时做同一件事。

**幂等**关注的是：

> 同一件事即使因为重试被执行多次，最终结果也不会不断产生新的副作用。

理想情况下，两者应该同时存在。

但一个锁本身不能让业务天然幂等。

例如：

\`\`\`text
先成功发送邮件
再记录“任务已完成”
\`\`\`

如果程序刚发送完邮件就崩溃，那么数据库里可能还没有完成记录。下一次重试仍然会发送第二封邮件。

所以更稳妥的设计是：

\`\`\`text
调度层：减少重复执行
业务层：设计幂等键
存储层：保存执行状态
外部副作用：尽可能支持幂等请求
\`\`\`

本文的 SQLite 锁只解决第一层和部分状态记录问题。

## 二、为什么这里可以使用 SQLite

SQLite 很适合单机自动化：

- Python 自带 \`sqlite3\`；
- 数据保存在一个文件里；
- 支持事务；
- 支持唯一约束；
- 多进程访问同一个数据库文件时可以由 SQLite 处理事务竞争。

关键点不是“SQLite 有一把神奇的锁”，而是我们把**任务键设置成唯一约束**，再用事务完成一次领取。

例如：

\`\`\`text
任务：daily-report
窗口：2026-10-07T00:00

实例 A → 尝试插入
实例 B → 同时尝试插入

只有一个实例能成功创建这条记录。
\`\`\`

## 三、完整可运行实现

下面保存为 \`idempotent_job.py\`。

它只使用 Python 3.10+ 标准库。

\`\`\`python
from __future__ import annotations

import argparse
import sqlite3
import sys
import time
from datetime import datetime, timezone
from pathlib import Path


SCHEMA = """
CREATE TABLE IF NOT EXISTS job_runs (
    job_key TEXT PRIMARY KEY,
    status TEXT NOT NULL CHECK (status IN ('running', 'success')),
    started_at TEXT NOT NULL,
    finished_at TEXT
)
"""


def connect(db_path: Path) -> sqlite3.Connection:
    connection = sqlite3.connect(
        db_path,
        timeout=5,
        isolation_level=None,
    )
    connection.execute("PRAGMA busy_timeout = 5000")
    connection.execute(SCHEMA)
    return connection


def claim_job(
    connection: sqlite3.Connection,
    job_key: str,
) -> bool:
    started_at = datetime.now(timezone.utc).isoformat()

    connection.execute("BEGIN IMMEDIATE")
    try:
        row = connection.execute(
            "SELECT status FROM job_runs WHERE job_key = ?",
            (job_key,),
        ).fetchone()

        if row is not None:
            connection.execute("ROLLBACK")
            return False

        connection.execute(
            """
            INSERT INTO job_runs(job_key, status, started_at)
            VALUES (?, 'running', ?)
            """,
            (job_key, started_at),
        )
        connection.execute("COMMIT")
        return True
    except Exception:
        connection.execute("ROLLBACK")
        raise


def finish_job(
    connection: sqlite3.Connection,
    job_key: str,
) -> None:
    finished_at = datetime.now(timezone.utc).isoformat()

    cursor = connection.execute(
        """
        UPDATE job_runs
        SET status = 'success', finished_at = ?
        WHERE job_key = ? AND status = 'running'
        """,
        (finished_at, job_key),
    )

    if cursor.rowcount != 1:
        raise RuntimeError(
            f"任务状态异常，无法完成：{job_key}"
        )


def release_job(
    connection: sqlite3.Connection,
    job_key: str,
) -> None:
    connection.execute(
        "DELETE FROM job_runs WHERE job_key = ? AND status = 'running'",
        (job_key,),
    )


def run_job(db_path: Path, job_key: str, seconds: float) -> int:
    connection = connect(db_path)

    try:
        if not claim_job(connection, job_key):
            print(f"跳过：任务已被领取或已经完成：{job_key}")
            return 3

        try:
            print(f"开始执行：{job_key}")
            time.sleep(seconds)
            finish_job(connection, job_key)
            print(f"执行完成：{job_key}")
            return 0
        except Exception:
            release_job(connection, job_key)
            raise
    finally:
        connection.close()


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Run a SQLite-backed idempotent local job."
    )
    parser.add_argument(
        "job_key",
        help="Unique key for one execution window",
    )
    parser.add_argument(
        "--db",
        default="job_runs.sqlite3",
        help="SQLite database path",
    )
    parser.add_argument(
        "--seconds",
        type=float,
        default=1.0,
        help="Simulated job duration",
    )
    args = parser.parse_args()

    if args.seconds < 0:
        print("错误：--seconds 不能小于 0", file=sys.stderr)
        return 2

    try:
        return run_job(
            Path(args.db),
            args.job_key,
            args.seconds,
        )
    except sqlite3.Error as exc:
        print(f"SQLite 错误：{exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
\`\`\`

运行：

\`\`\`bash
python idempotent_job.py daily-report
\`\`\`

第一次执行会进入“开始执行 → 执行完成”，再次使用相同任务键会被跳过。

程序用退出码区分结果：

\`\`\`text
0 = 本次成功执行
3 = 已被其他实例领取或已经完成
2 = 参数错误
1 = SQLite 或其他执行错误
\`\`\`

## 四、为什么不用“先查询，再插入”

下面这种写法看起来很直观：

\`\`\`python
if not exists(job_key):
    insert(job_key)
\`\`\`

但它存在竞态：

\`\`\`text
实例 A：查询 → 不存在
实例 B：查询 → 不存在

实例 A：插入
实例 B：插入
\`\`\`

两个实例都通过了检查。

所以真正重要的是：

\`\`\`text
查询 + 决策 + 写入
\`\`\`

必须放在受事务保护的临界区里。

本文使用：

\`\`\`python
connection.execute("BEGIN IMMEDIATE")
\`\`\`

让当前 SQLite 事务先取得写事务所需的锁，然后检查并插入。

同时数据库还有：

\`\`\`sql
job_key TEXT PRIMARY KEY
\`\`\`

即使未来代码发生变化，数据库本身仍然有唯一性约束。

这就是一个很重要的工程原则：

> **关键不变量不要只写在代码里，也应该尽可能写进数据约束。**

## 五、为什么失败时要释放 running 状态

假设任务领取成功后发生异常：

\`\`\`text
claim
 ↓
running
 ↓
执行任务
 ↓
异常
\`\`\`

如果程序什么都不做，那么下一次运行看到：

\`\`\`text
status = running
\`\`\`

可能误以为任务还在执行。

本文的示例在异常路径调用：

\`\`\`python
release_job(...)
\`\`\`

把这次未完成的领取删除，让下一次执行重新获得机会。

但这里仍然有一个边界：

**如果进程直接被 kill，或者机器突然断电，Python 没机会执行 \`release_job()\`。**

所以生产环境不能只依赖这个简单版本。

## 六、真正生产环境还需要“租约”

更完整的设计通常会增加：

\`\`\`text
job_key
owner_id
started_at
expires_at
heartbeat
status
\`\`\`

例如：

\`\`\`text
running
   ↓
正常完成 → success

running
   ↓
超过 expires_at
   ↓
允许其他实例重新领取
\`\`\`

这样可以解决：

> “锁的持有者已经死了，但数据库还认为它正在运行。”

不过这已经进入租约、心跳和故障恢复的问题。

如果任务跨机器运行，还要进一步考虑：

- 数据库是否被所有实例可靠访问；
- 网络分区；
- 时钟问题；
- 锁续租；
- 进程崩溃；
- 外部副作用重复；
- 数据库本身的高可用。

因此不能把这个几十行的 SQLite 示例包装成“分布式任务调度系统”。

## 七、测试：不要只测正常情况

至少应该测试三种情况。

### 1. 第一次领取

\`\`\`bash
python idempotent_job.py test-001
\`\`\`

应该返回：

\`\`\`text
0
\`\`\`

### 2. 相同任务键再次运行

\`\`\`bash
python idempotent_job.py test-001
\`\`\`

应该返回：

\`\`\`text
3
\`\`\`

### 3. 参数错误

\`\`\`bash
python idempotent_job.py test-002 --seconds -1
\`\`\`

应该返回：

\`\`\`text
2
\`\`\`

还可以让两个终端同时执行相同任务键：

\`\`\`bash
python idempotent_job.py concurrent-test --seconds 3
python idempotent_job.py concurrent-test --seconds 3
\`\`\`

理想结果是一个实例进入“开始执行”，另一个实例进入“跳过”。

这里真正值得验证的不是“代码能不能启动”，而是**两个实例竞争时是否仍然满足唯一领取的不变量**。

## 八、这个方案适合什么，不适合什么

适合：

- 单机定时脚本；
- 本地数据处理；
- 每日生成报告；
- 小型自动化任务；
- 需要保存任务执行状态的工具；
- 不想额外部署 Redis 等服务的场景。

不适合直接拿来解决：

- 多机高可用调度；
- 大规模分布式任务；
- 跨地域任务锁；
- 强一致的金融交易；
- 复杂工作流编排。

如果业务真的进入这些场景，就应该根据实际需求选择数据库锁、消息队列、任务调度系统或专门的分布式协调机制。

## 九、把它放回真实自动化系统

一个比较可靠的自动化任务，不应该只有：

\`\`\`text
定时器 → 执行
\`\`\`

而应该逐渐变成：

\`\`\`text
定时触发
   ↓
读取最新状态
   ↓
计算幂等键
   ↓
尝试领取执行权
   ↓
质量检查
   ↓
执行副作用
   ↓
记录结果
   ↓
发布后验证
\`\`\`

其中最容易被忽略的是：

\`\`\`text
计算幂等键
\`\`\`

例如一个“每 12 小时最多执行一次”的任务，可以把窗口设计成：

\`\`\`text
2026-10-07T00
2026-10-07T12
2026-10-08T00
\`\`\`

任务键可以由：

\`\`\`text
任务名称 + 时间窗口
\`\`\`

构成。

这样调度器即使因为重试触发两次，也不会天然产生两次业务执行。

## 十、结论

自动化系统真正难的地方，往往不是“让它跑起来”，而是让它在：

- 重试；
- 并发；
- 崩溃；
- 超时；
- 网络异常；
- 重复触发

这些真实情况下仍然保持可控。

SQLite 可以用很低的成本解决一部分问题，但它不是万能锁。

最值得保留的设计思路其实只有三条：

1. **为一次业务执行定义明确的幂等键。**
2. **把关键唯一性放进数据库约束和事务。**
3. **把“防重复执行”和“业务本身幂等”分开设计。**

当自动化任务开始产生真实副作用时，这三条往往比“把定时器设置得更聪明”更重要。
