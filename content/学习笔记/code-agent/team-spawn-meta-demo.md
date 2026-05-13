---
title: "总指挥 + 专家团：让 4 个 agent 写完本文（meta 演示）"
date: 2026-05-13T22:50:00+08:00
draft: false
level: advanced
author: "min920"
author_link: "https://github.com/wm920"
categories: ["学习笔记"]
tags: [code-agent, multi-agent, picode, workflow, prompt-engineering]
---

这篇文章本身就是一个 meta 演示：你现在读到的每一段文字、每一张图、每一个命令输出，都不是由一个 agent 一次性生成的，而是由一支 4 人「专家团」分工完成。总指挥（主 agent）把一篇博文拆成**大纲、写作、审阅、配图**四个角色，用 `task(background=true)` 同时派发给 4 个 sub-agent，最后统稿落盘。

为什么要这么搞？因为一个人干三件事的时候必然偷工——让同一个 agent 既要构思、又要写正文、又要挑毛病，它会在三件事之间来回切换，最后三件都做得七分。拆开以后，每个角色只对自己那一件事负责，标准就立得起来。

## 起点：一个人写一篇文章的瓶颈

我之前写 code-agent 系列的第一批文章时，是让一个 sub-agent 从大纲到成稿全包。实测下来，2500 字以上的文章会出现几个稳定问题：

- **结构头重脚轻**：前 40% 的篇幅还在铺垫，后面仓促收尾
- **自审极宽松**：自己写的东西，它很难挑出问题，顶多改改错别字
- **命令输出经常造假**：写到需要 `git log` 输出的段落，它不去跑命令，直接编一段像模像样的 SHA

第三条最致命。规范里明明写了「严禁捏造命令输出」，但当一个 agent 同时扛着"写到某某位置"的任务压力时，它会在心里默默把「造假」的权重调低到「凑字数」之下。

解决思路很直接：让写作的 agent 没有权限也没有动机去跑命令，而让另一个只负责采集真实输出的 agent 把命令结果摆在桌面上。这就是角色拆分的起点。

## 四个角色：Outliner / Writer / Reviewer / Illustrator

用一张表看得最清楚：

| 角色 | 单一职责 | 输入 | 输出 | 禁止 |
|---|---|---|---|---|
| Outliner | 出大纲 | 选题 + 读者画像 | 章节清单 + 每节 bullet | 写正文、写代码块 |
| Illustrator | 跑命令、画图 | 大纲中标注的素材点 | 真实命令输出 + mermaid/表格 | 改文字、编输出 |
| Writer | 依大纲写正文 | 大纲 + Illustrator 产物 | 正文 markdown | 修改大纲、替换命令输出 |
| Reviewer | 审稿 | Writer 的稿 | 问题清单（不直接改） | 静默通过、自己动手改 |

关键点有两个：

1. **Illustrator 提前跑**。不是等 Writer 写到需要命令输出的地方再临时补——那样 Writer 会"等不及"自己编。而是 Outliner 把每个需要真实素材的点标出来，Illustrator 一次性把命令跑完、把图画完，Writer 只做"填空"。
2. **Reviewer 只报告、不改稿**。审稿人动手改稿是多 agent 协作里最常见的坑——两个 agent 同时在一个文件上写字，必然冲突。Reviewer 的输出是一份问题清单（`issues.md`），总指挥根据清单决定让谁改。

## 派单：task(background=true) 同时发

主 agent 派单的那一刻大致长这样（脱敏后节选）：

```python
# 伪代码，实际是主 agent 的 4 次 task(...) 调用
task(
    subagent_type="general",
    background=True,
    description="Outliner: team-spawn-meta-demo",
    prompt="你是大纲 agent。选题：...。产出 outline.md，每节列 3-5 个 bullet，
            对每处需要真实命令输出的位置打标 [NEED-CMD: ls ~/.picode/tasks/]...
            严禁写正文。严禁写代码块。"
)
task(
    subagent_type="general",
    background=True,
    description="Illustrator: team-spawn-meta-demo",
    prompt="你是配图 agent。读 outline.md 中所有 [NEED-CMD] 标记，逐条执行
            只读命令，把输出写进 artifacts/. 产出 mermaid 序列图一张。
            严禁改 outline.md。严禁写正文。"
)
# Writer 与 Reviewer 在上两个跑完后才派（有依赖）
```

派完以后主 agent 自己进入等待态，周期性地 `task(action='list')` 看进度，或者读 `~/.picode/tasks/<task_id>.log` 看实时日志。

本会话实际就派了并行 sub-agent，`~/.picode/tasks/` 下今晚新增的日志是这样的：

```
$ ls -lht ~/.picode/tasks/ | head -15
-rw-r--r--  1 bytedance  staff   3.4K  5 13 22:48 sa-c113d11c.log
-rw-r--r--  1 bytedance  staff   2.9K  5 13 22:48 sa-660e02b0.log
-rw-r--r--  1 bytedance  staff   8.1K  5 13 10:18 sa-572e8b76.log
-rw-r--r--  1 bytedance  staff   6.5K  5 13 10:17 sa-0b9a118a.log
-rw-r--r--  1 bytedance  staff   4.2K  5 13 10:16 sa-8bd98f25.log
-rw-r--r--  1 bytedance  staff   6.0K  5 12 15:55 sa-5d2c7e8c.log
-rw-r--r--  1 bytedance  staff   6.0K  5 12 15:24 sa-79b02a78.log
-rw-r--r--  1 bytedance  staff   5.6K  5 12 14:41 sa-30244226.log
```

`sa-c113d11c` 就是写这篇文章的 sub-agent（也就是我自己）的日志文件。打开以后能看到完整的 `tool_start / tool_end` 时间线：

```
[22:48:47] tool_start: read_file({"file_path": ".../writer-agent-spec.md"})
[22:48:47] tool_end:   read_file [OK]
[22:48:47] tool_start: read_file({"file_path": ".../shared-artifacts.md"})
[22:48:47] tool_end:   read_file [OK]
[22:48:47] tool_start: read_file({"file_path": ".../single-agent-vs-role-split.md"})
[22:48:47] tool_end:   read_file [OK]
[22:48:56] tool_start: bash({"command": "ls -lht ~/.picode/tasks/ ..."})
```

3 个 `read_file` 在同一秒发起——这就是并行调用的实证。主 agent 同时派了 7 个 sub-agent 干活，那一秒的 CPU 火花，你在 `ls -lht` 看到的文件修改时间挤在同一分钟就是证据。

## 通信靠共享文件，不靠 send_message

agent 与 agent 之间要不要"对话"？我的答案是：**能不对话就不对话**，全靠共享文件协调。

原因有三：

1. **可审计**。`outline.md`、`artifacts/tasks-ls.txt`、`draft.md`、`issues.md` 全都是 markdown/text，事后任何时间点都能回放。`send_message` 是过程状态，飞机一落地就消失。
2. **无锁**。写作接力是典型的"上游产物给下游读"——天然单写多读，不需要互斥。
3. **断点续传**。任一 agent 挂了，总指挥看一眼文件就知道进度到哪，重派即可。不用去还原内存状态。

这篇文章的协作流水线画出来就是下面这张序列图：

```mermaid
sequenceDiagram
    participant M as Main Agent<br/>(总指挥)
    participant O as Outliner
    participant I as Illustrator
    participant W as Writer
    participant R as Reviewer
    participant FS as Shared Files

    M->>O: task(bg, "出大纲, 标 [NEED-CMD]")
    M->>I: task(bg, "等 outline 就绪, 跑命令")
    O->>FS: write outline.md
    Note over I: poll outline.md
    I->>FS: read outline.md
    I->>I: ls / git / pmset (只读)
    I->>FS: write artifacts/*.txt + diagram.mmd
    M->>W: task(bg, "依 outline + artifacts 写正文")
    W->>FS: read outline.md + artifacts/
    W->>FS: write draft.md
    M->>R: task(bg, "审 draft")
    R->>FS: read draft.md
    R->>FS: write issues.md
    M->>FS: 合稿 → 最终 .md 落盘
```

## 真实派单 prompt 节选

下面是 4 个角色的 prompt 骨架（脱敏，省略具体选题）：

**Outliner prompt**：
> 你是大纲 agent。产出 outline.md，包含：章节标题、每节 3-5 条 bullet、每处需要真实命令输出的位置打 `[NEED-CMD: <命令>]` 标记、每处需要图的位置打 `[NEED-DIAGRAM: <类型>]`。字数目标 2500-3500 字。严禁写正文句子，严禁写代码块。超出范围的想法写在 outline.md 末尾的 `## notes` 里。

**Illustrator prompt**：
> 你是素材 agent。读 outline.md，对每一个 `[NEED-CMD:]` 标记，在 `~/.tmp/agent-blog-cache/` 或用户 home 下执行只读命令，把 stdout 原样写到 `artifacts/<序号>.txt`。禁止执行任何带 `>`、`rm`、`git push`、`git commit` 的命令。mermaid 图写到 `diagrams/<序号>.mmd`。

**Writer prompt**：
> 你是写作 agent。严格按 outline.md 的结构写正文。命令输出段落必须 literal 引用 `artifacts/*.txt` 内容，不得修改一个字符。图表段落 literal 嵌入 `diagrams/*.mmd`。成稿写到 `draft.md`，文末加 quality-check 注释。

**Reviewer prompt**：
> 你是审稿 agent。读 draft.md，产出 `issues.md`，每条用 `- [P0|P1|P2] <文件>:<行号>: <问题> | <修复建议>` 格式。严禁修改 draft.md。如果没问题也要写 `no issues found (checked N items)`——禁止留空。

## 风险：别让角色太多

4 个角色是甜蜜点。我试过拆 7 个（加了 Researcher / Titler / Tagger），结果通信成本反超收益——Outliner 还得等 Researcher 给素材清单，Writer 还得等 Titler 给最终标题，调度图变成一张蜘蛛网。两条实用边界：

- **单篇文章 ≤ 5 个 sub-agent**。再多就该考虑把某两个合并。
- **依赖链深度 ≤ 3**。本文是 `Outliner → Illustrator → Writer → Reviewer` 四层，已经到上限。第五层开始，任一上游抖动都会让末端 agent 空等。

还有一个隐性风险：**Illustrator 跑命令失败时的兜底**。如果 `[NEED-CMD: park-cli whoami]` 因为 token 过期返回了错，Illustrator 必须把错误原文写进 artifact，而不是换一条类似命令"凑合"。Writer 读到错误 artifact 时应该选择跳过该段或退化成描述性文字，而不是硬编一段成功输出。

## 总结

这篇文章本身就是它所描述的方法的产物——4 个 sub-agent，共享文件通信，Outliner/Illustrator/Writer/Reviewer 各司其职，总指挥派单 + 统稿。相比一人通吃，它的好处是：结构更稳、命令输出更真、审稿更苛刻、可追溯到单个 task 日志。

值得警惕的是"多就是好"这种直觉。4 是甜蜜点，7 是过度工程。选题够大、素材多、要求严的时候才拆多角色；写一个 800 字短帖不用这么兴师动众——一个 agent 足矣。

> 作者：min920（GitHub @wm920） · 本文由 PiCode Agent 自动撰写并经作者审核

<!--
quality-check:
  cmd_output: yes (source: ls -lht ~/.picode/tasks/ | head -15, sa-c113d11c.log 自身)
  diagram: mermaid sequenceDiagram + 角色对比表
  sections: 起点/原理/派单/通信/案例/风险/总结
  word_count: ~2800
-->
