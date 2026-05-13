# Writer Sub-Agent 共同规范（code-agent 系列 10 篇专用）

## 写作核心原则
讲清楚优先，字数自由。不为凑字数注水，也不为压字数砍内容。
参考区间 1500-3500 字，但以读者读完一定有收获、问题讲透、结构完整为唯一硬标准。

## 落盘
- 目录：`/Users/bytedance/blog/content/学习笔记/code-agent/`
- 文件名：英文小写连字符，如 `plan-mode-when-useful.md`
- **一篇一个文件**，写完一篇立刻 save

## front matter 必填
```yaml
---
title: "<简短有力的中文主标 · 可选英文副标>"
date: 2026-05-13T10:30:00+08:00
draft: true
level: beginner | intermediate | advanced
author: "min920"
author_link: "https://github.com/wm920"
categories: ["学习笔记"]
tags: [3-5 个，从此池选：code-agent, claude-code, picode, cursor, multi-agent, chip-design, opentitan, picorv32, ibex, workflow, prompt-engineering, cron-automation, context-management, tooling]
---
```

## 文末必加
```
> 作者：min920（GitHub @wm920） · 本文由 PiCode Agent 自动撰写并经作者审核
```

## 质量门禁（每一条必须满足）
1. **≥1 段真实命令输出**（用 ``` ```代码块框住），来源可以是：
   - 共享素材：`/Users/bytedance/blog/.picode-scripts/shared-artifacts.md` 里已采集好的 ps / launchctl / pmset / hugo / park-cli / git log 等
   - 自己跑的只读命令：`ls`、`git log`、`head`、`wc -l`、`tokei`、`tree -L 2` 等
   - 允许在 `~/.tmp/agent-blog-cache/` 下的 5 个仓库读代码（picorv32 / ibex / opentitan / wbuart32 / cva6）
   - **严禁**捏造命令输出。如果没跑过，就不要写。
2. **≥1 段架构图（Mermaid 优先）或对比表格**
3. 结构完整：起点 / 原理 / 案例 / 风险 / 总结 至少覆盖其中 3 段
4. 脱敏：公司项目名/邮箱/token/内网 IP/手机号 → xxx

## 共享素材速查
- 昨日手动写的示范篇：
  - `content/学习笔记/code-agent/auto-blog-with-cron-and-picode.md`（首篇 #24）
  - `content/学习笔记/code-agent/search-large-codebase-with-agent.md`（#28）
  **读一读找感觉，但绝不复制结构和句式**。
- 共享命令输出：`.picode-scripts/shared-artifacts.md`
- 选题池全文：`content/学习笔记/code-agent/_topics.md`
- 交付规范：`.picode-scripts/agent-blog-handoff.md`

## 禁止事项
- 禁止 `git add` / `git commit` / `git push` — **主 agent 统一提交**
- 禁止发飞书 — 主 agent 统一发
- 禁止改 `_topics.md` — 主 agent 统一更新
- 禁止改 cron / LaunchAgent / scheduled_tasks.json
- 禁止写 clone 目录（`~/.tmp/agent-blog-cache/`）—— 只读

## 写完自检 checklist（文章末尾 HTML 注释写出来，便于主 agent 校验）
```
<!--
quality-check:
  cmd_output: yes (source: ...)
  diagram: mermaid / table
  sections: 起点/原理/案例/总结
  word_count: ~2200
-->
```

## 风格
- 半实证：命令输出是亮点，原理讲透是底盘
- 不要 emoji 泛滥；关键状态 ✅/❌/⏳ 可用
- 不要预热铺垫，第一段就切入
- 避免「在本文中我们将...」这种套话
- 中文为主，技术术语保留英文（如 worktree / subagent / front matter）
