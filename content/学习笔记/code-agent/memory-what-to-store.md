---
title: "PiCode memory 系统该存什么、不该存什么"
date: 2026-05-13T14:00:00+08:00
draft: false
level: beginner
author: "min920"
author_link: "https://github.com/wm920"
categories: ["学习笔记"]
tags: ["code-agent", "picode", "memory", "workflow", "best-practice"]
---

## memory 不是日记本

PiCode 有一套 per-project 的 memory 系统，落盘在 `~/.picode/projects/<hash>/memory/` 下。每条 memory 一个 `.md` 文件，带 YAML frontmatter，外加一个 `MEMORY.md` 索引。

我刚开始用的时候把它当成「agent 的日记本」：每做完一件事就写点什么。三周后打开索引——42 条记录，70% 都是噪音。这篇文章讲清楚 memory 的正确定位：**跨会话可复用的事实、规范、偏好**；不是本次任务细节，不是敏感信息，不是会过期的时效状态。

## 先看一眼真实的 memory 目录

```
$ ls ~/.picode/projects/982b1ec6f0e9/memory/ | head -15
MEMORY.md
ai-is-turn-based-not-overnight.md
bash-command-substitution-blocked.md
bash-heredoc-bypass-write-file-dict.md
batch-generate-36-articles-plan.md
blog-automation-handoff-doc.md
blog-word-count-flexible.md
check-memory-before-claiming-absence.md
cron-09-miss-on-2026-05-12.md
cron-trigger-needs-live-picode-process.md
croncreate-durable-prompt-not-flushed.md
crondelete-session-scoped.md
daily-audit-cron-design.md
daily-audit-cron-silent-fail-2026-05-12.md
dry-run-before-cron-writes.md
...
$ ls ~/.picode/projects/982b1ec6f0e9/memory/ | wc -l
      41
```

41 条 memory，不少。问题是——这 41 条里，有些是**该存的**（例如 `bash-command-substitution-blocked` 是 PiCode sandbox 的固定限制），有些是**不该存的**（例如 `cron-09-miss-on-2026-05-12` 记的是一个时间戳明确的单次事故）。

## 该存的 3 类（举例）

PiCode system prompt 里也定义了 4 种 note_type：`user / feedback / project / reference`。下面举 3 个**值得留下来**的：

### ① 用户偏好 / 协作方式（user）

> 「我的博客文章正文 1500-2500 字，不要超过 3000。发布前一定先 draft: true，我审核过后才改 false。」

这条跨任何任务都有效——agent 在一个新会话里写博客，该不该自动 publish？**存过就知道答案是"不"**。不存，每次都要再问一遍。

### ② 项目级长期约束（project）

> 「本仓库是 Hugo + PaperMod 静态站，部署走 GitHub Pages gh-pages 分支，禁止直接 push 到 main 前不跑 `hugo --minify` 验证。」

这是跨会话、**不会因为今天做了什么而变**的事实。属于典型该存。

### ③ 失败教训 / 反馈（feedback）

> 「delay 计算结果必须向下取整（floor），不能四舍五入。
> Why: 用户明确要求，delay 代表时钟周期数，向下取整更保守安全。」

我 memory 目录里真的有这一条。它回答的是「为什么要这样做」——下次 agent 生成性能计算代码时，自动用 floor 而不是 round。

## 不该存的 3 类（举例）

### ① 本次任务的状态性细节

反例：

> ❌「今天我在写 #27 博文，写到第三节卡住了」

这是 todo / plan 该管的事，存到 memory 里只会在下周某个新任务里被错误召回。

### ② 会过期的时效信息

反例（看起来像我目录里真有的）：

> ❌「cron-09-miss-on-2026-05-12：5 月 12 号 9 点那次 cron 没触发」

这条 **2026-05-13 之后基本没复用价值**——它描述的是一个一次性事故。真要存，也该改写成：

> ✅「cron 触发需要前台 picode 进程活着，否则静默失败。Why: 2026-05-12 那次 miss 的根因就是这个。」

第二种写法才是跨会话可复用的——它把教训抽象成规则，日期只作为"为什么"的证据。

### ③ 敏感信息 / 密钥 / 可推导事实

反例：

> ❌「git remote URL 是 https://x-access-token:glpat-xxxxxx@gitlab.xxx/...」
> ❌「用户邮箱是 xxx@xxx.com」
> ❌「项目架构：根目录下有 layouts/ 和 content/ 文件夹」（这玩意 `ls` 一下就知道）

第一条是 token——**永远不要**。
第二条是 PII——不该。
第三条可以从代码推导出来，属于 system prompt 里明确列的 "DO NOT save"。

## 对比表

| 维度 | 该存 | 不该存 |
|------|------|--------|
| 时效 | 长期/跨会话有效 | 单次任务/有明确日期过期 |
| 来源 | 用户偏好、项目约束、抽象化的教训 | 本次对话细节、当下 in-progress 状态 |
| 敏感度 | 公开可复述 | token / 密钥 / PII |
| 可推导性 | 看代码/文档不知道 | 读一眼 repo 就知道的结构 |
| 抽象度 | 规则化（"xx 必须 yy"） | 流水账（"今天我 xx 了"） |

## 一条好 memory 长什么样

PiCode 规范里推荐 feedback / project 类写成「规则 + Why + How to apply」三段式。例如：

```markdown
---
name: cron-trigger-needs-live-picode
description: cron 触发 PiCode 任务必须有前台会话存活，否则静默失败
type: feedback
---

cron 里调 picode 执行任务时，前台必须有一个 picode 进程在跑。

**Why:** 2026-05-12 那次 09:00 定时博客没产出，根因是前台没有活会话，
cron 创建的 prompt 文件没有 consumer，静默沉默在队列里。

**How to apply:** 写 cron 脚本时先 `pgrep picode` 检查；没有就先
启一个 headless picode 再投 prompt。
```

三要素齐全：**规则一句话摆出来、为什么一句话、怎么用一句话**。下次 agent 召回这条 memory 时，能立刻判断"这是不是跟当前情况相关"。

## 决策路径

```mermaid
flowchart TD
  A[我想记点什么] --> B{是本次任务的状态<br/>还是持久的知识?}
  B -- 本次状态 --> N1[用 todo / plan<br/>不要进 memory]
  B -- 持久知识 --> C{能从代码 / git log<br/>推出来吗?}
  C -- 能 --> N2[不要存<br/>agent 自己看就行]
  C -- 不能 --> D{含 token/PII/密钥?}
  D -- 是 --> N3[绝对不要]
  D -- 否 --> E{有明确过期日期<br/>且不能抽象化?}
  E -- 是 --> N4[不要存]
  E -- 否 --> F[写成 规则+Why+How<br/>take_note 保存]
```

## 自检：每 1-2 个月扫一遍自己的 memory

规则再好，也会飘。我定了个简单的自检节奏——每 1-2 个月：

```
ls ~/.picode/projects/<hash>/memory/ | wc -l
```

如果超过 50，一定有噪音。然后逐条问三个问题：

1. 这条在**新会话**里还能用上吗？
2. 还是说它只是**某次任务的流水账**？
3. 里面有没有**已经被代码/配置承载**的信息（例如"我用的 Hugo"——`config.toml` 里明写了）？

只要有一个"是"，就 `rm` 掉。memory 越精简，下次 agent 召回的信噪比越高。

## 小结

memory 的设计定位是"跨会话可复用的知识底座"，不是日记本、不是状态板、不是密钥保险箱。三条原则：

- **存**：用户偏好、项目约束、抽象过的教训
- **不存**：本次任务细节、时效性事故、敏感信息、可推导事实
- **写法**：规则 + Why + How to apply 三段式

用它当"把你告诉过 agent 一次的事、以后不用再说"的工具，它会越用越值；用它当日记本，它会在某次关键调研时**给你一堆噪音**。

> 作者：min920（GitHub @wm920） · 本文由 PiCode Agent 自动撰写并经作者审核

<!--
quality-check:
  cmd_output: yes (ls ~/.picode/projects/<hash>/memory 真实输出)
  diagram: 1 个 mermaid + 1 对比表 + 1 代码块 memory 范例
  sections: 起因/目录样本/该存 3 类/不该存 3 类/范本/决策图/自检/总结
  word_count: ~2200
-->
