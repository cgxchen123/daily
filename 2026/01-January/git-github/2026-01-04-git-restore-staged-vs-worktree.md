---
title: "撤销 Git 修改前，先分清暂存区和工作区"
date: 2026-01-04
category: git-github
tags: [git, version-control, workflow]
difficulty: intermediate
reading_time: 6 min
python_version: "3.10+"
dependencies: []
series: null
---

# 撤销 Git 修改前，先分清暂存区和工作区

Git 中“修改了文件”“已暂存”和“已提交”是三个不同状态。遇到误操作时，如果不先看状态就执行重置命令，可能把还没提交的工作一起丢掉。

## 先查看状态，再选择操作

```bash
git status --short
git diff
git diff --staged
```

这三条命令分别帮助你识别文件状态、查看工作区未暂存差异，以及查看暂存区将进入下一次提交的内容。提交前，尤其要检查新增文件是否含有密钥或临时数据。

## 常见操作的区别

- `git restore <file>`：用索引中的版本覆盖工作区文件，丢弃该文件未暂存的修改。
- `git restore --staged <file>`：把文件从暂存区移出，但保留工作区内容。
- `git diff --staged`：查看暂存区与当前提交之间的差异。
- `git log --oneline -5`：快速查看最近提交。

不要把 `git reset --hard` 当作通用的“修复命令”。它会重置索引和工作区，可能造成未提交修改丢失。

## 推荐的提交前流程

```bash
git status --short
git diff --check
git diff --staged
```

`git diff --check` 可以发现部分空白字符问题，但它不会判断业务逻辑是否正确。代码审查仍需要阅读实际差异、运行测试，并确认提交范围。

## 发生误操作怎么办

先停止继续写入，运行 `git status` 查看当前状态。若内容重要，优先复制工作区文件到仓库外备份，再考虑恢复操作。已经提交的内容通常可以通过新提交修正；不要为了让历史“看起来干净”就随意改写共享分支历史。

核心习惯很简单：每次执行破坏性 Git 命令前，先确认它影响哪个状态，以及是否有可恢复的副本。
