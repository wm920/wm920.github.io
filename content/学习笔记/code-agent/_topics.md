---
title: "_topics"
draft: true
_build:
  render: never
  list: never
  publishResources: false
---

# Code Agent 博客选题池（agent 内部使用，draft）

最后更新：2026-05-13（全量清池：36 篇全部落盘，D 储备池待启用）

落盘目录：`content/学习笔记/code-agent/`
分级字段 front matter：`level: beginner | intermediate | advanced`

---

## A. 多角色 Agent（10）

1. ✅ [intermediate] 单 agent vs 角色拆分：什么时候该拆 — `single-agent-vs-role-split.md`
2. ✅ [advanced] 总指挥 + 专家团：让 4 个 agent 写完本文（meta 演示，team_spawn 实跑） — `team-spawn-meta-demo.md`
3. ✅ [advanced] 给角色写 system prompt 的 5 条戒律 — `role-system-prompt-5-rules.md`
4. ✅ [intermediate] 共享 task list vs send_message：协作通信怎么选 — `shared-tasklist-vs-sendmessage.md`
5. ✅ [advanced] git worktree 隔离避免多 agent 改同一文件 — `git-worktree-multi-agent.md`
6. ✅ [advanced] Reviewer 角色实战：让一个 agent 专挑另一个的毛病 — `reviewer-agent-in-practice.md`
7. ✅ [intermediate] DevOps 角色：专门跑 CI + 报失败的 agent — `devops-agent-ci-runner.md`
8. ✅ [intermediate] 文档 Agent：代码改完自动同步 README — `docs-agent-auto-sync-readme.md`
9. ✅ [advanced] commit 前的质量门禁 Agent — `pre-commit-quality-gate-agent.md`
10. ✅ [intermediate] 多角色协作的反模式：哪些活不该拆 — `multi-agent-anti-patterns.md`

## B. 芯片项目长期维护（13）

仓库白名单（允许 `git clone --depth=1` 到 `~/.tmp/agent-blog-cache/`，禁止写回）：
- PicoRV32  https://github.com/YosysHQ/picorv32
- Ibex      https://github.com/lowRISC/ibex
- OpenTitan https://github.com/lowRISC/opentitan
- wbuart32  https://github.com/ZipCPU/wbuart32
- CVA6      https://github.com/openhwgroup/cva6

11. ✅ [beginner] 用 agent 5 分钟读完 PicoRV32 — `read-picorv32-in-5-min.md`
12. ✅ [intermediate] 让 agent 给 PicoRV32 加一条自定义指令（端到端） — `add-custom-instruction-picorv32.md`
13. ✅ [intermediate] agent 读寄存器手册自动生成 register file（OpenTitan 风格） — `register-file-from-manual.md`
14. ✅ [advanced] Ibex 代码评审 agent：按 lowRISC coding style 自动 review — `ibex-review-agent-lowrisc-style.md`
15. ✅ [intermediate] 波形 / log 分析 agent：仿真失败自动定位 — `waveform-log-analysis-agent.md`
16. ✅ [advanced] UVM 测试用例自动补全：覆盖率驱动 agent 补 case — `uvm-case-auto-gen-by-coverage.md`
17. ✅ [intermediate] Verilator + agent：开源工具链怎么串起来 — `verilator-agent-toolchain.md`
18. ✅ [intermediate] 给 wbuart32 加一个新外设：agent 主导的实战 — `add-peripheral-to-wbuart32.md`
19. ✅ [advanced] RTL 改了自动更新设计文档（飞书 wiki 同步） — `rtl-change-doc-sync-feishu.md`
20. ✅ [advanced] Bug 长期追踪 agent：commit 引入 bug 自动指认 — `bug-bisect-agent.md`
21. ✅ [intermediate] 新人 onboarding agent：带新人逛 OpenTitan — `onboarding-agent-opentitan-tour.md`
22. ✅ [intermediate] 多版本分支并行维护：agent 自动 cherry-pick — `multi-branch-cherry-pick-agent.md`
23. ✅ [advanced] 为什么传统 EDA 工作流难以 agent 化（元话题） — `why-eda-hard-to-agentize.md`

## C. 通用实战 / 工作流（13）

24. ✅ [beginner] 用 cron + PiCode 让 agent 每天写博客 — `auto-blog-with-cron-and-picode.md`
25. ✅ [beginner] Claude Code 的 plan mode 什么时候真有用 — `plan-mode-when-useful.md`
26. ✅ [intermediate] PiCode 的 use_skill 按需加载机制 — `picode-use-skill-lazy-load.md`
27. ✅ [beginner] todo_write 用对的姿势 — `todo-write-right-way.md`
28. ✅ [intermediate] 大代码库检索三件套 — `search-large-codebase-with-agent.md`
29. ✅ [intermediate] 写好 picode.md / CLAUDE.md 的几条经验 — `picode-md-claude-md-practices.md`
30. ✅ [beginner] memory 系统该存什么、不该存什么 — `memory-what-to-store.md`
31. ✅ [intermediate] background bash 在长跑任务里的用法 — `background-bash-for-long-tasks.md`
32. ✅ [intermediate] 群聊驱动的 debug 工作流实测 — `chat-driven-debug-workflow.md`
33. ✅ [intermediate] Agent 做 GitLab MR Code Review 的工作流 — `gitlab-mr-review-agent.md`
34. ✅ [beginner] 让 agent 写出靠谱 commit message 的提示词 — `commit-message-prompt-for-agent.md`
35. ✅ [intermediate] "AI 帮我看 CI 失败"完整复盘 — `ai-debug-ci-failure.md`
36. ✅ [intermediate] Mermaid 一键转飞书画板自动化 — `mermaid-to-feishu-board.md`

## D. 储备池（主池已 100% 清空，待启用）

下次 Code Agent cron 任务 fire 时，应触发 `web_fetch` 抓近期 Claude Code / Agent 领域 changelog 或博客，识别"未覆盖角度"后追加。候选方向：tool calling 协议设计、context window 压缩原理、长上下文 vs RAG、agent 写测试是否可信、本地模型替换 Claude、proprietary 代码风险、跨语种代码迁移、agent 沉浸式调试。

---

**注**：本文件 front matter 不含 draft，但开头是普通 markdown 不会被 Hugo 当成文章（无 front matter）。如担心被列入站点，写一行 `+++` 头加 `draft: true` 或重命名为 `.md.txt`。
