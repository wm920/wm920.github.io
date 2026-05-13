# 「AI Agent 博客自动化系统」完整交付规范

> 目标：让另一个 AI Agent（Claude Code / PiCode / Cursor 等）从 0 搭起一套**每天自动写技术博客**的系统。本文件是一次性交付，另一端读完能直接落地。

## 用户画像与初始诉求

- 用户：芯片 / 互联协议领域工程师（GitHub @wm920，飞书登录邮箱已配）
- 已有：Hugo 博客仓库 `~/blog`，主仓库 main 分支，站点 wm920.github.io
- 诉求：让 agent 每天**自动**选题、写技术博文、发到 Hugo 博客、推 GitHub、发飞书摘要通知
- 内容方向：
  - Code Agent / Claude Code / PiCode 用法（**重点**）
  - 多角色 agent 协作
  - Agent 长期维护一个芯片项目（用开源 repo 举例：PicoRV32/Ibex/OpenTitan/wbuart32/CVA6）
- 写作核心原则：**讲清楚优先，字数自由**——不为凑字数注水，也不为压字数砍内容
- 风格：半实证——允许跑只读命令把真实输出截进文章（敏感信息脱敏为 xxx）
- 部署机：macOS（Apple Silicon）

## 内容体系设计

### 三大选题类别 + 分级

```
A. 多角色 Agent（每周 1-2 篇）
B. 芯片项目长期维护（每周 1-2 篇）
C. 通用实战 / 工作流（每周 1-2 篇）

level: beginner | intermediate | advanced
难度滚动，避免连续 3 篇 advanced
```

### 选题池外置文件（必须）

路径：`content/学习笔记/code-agent/_topics.md`（front matter 设 `draft: true` + `_build: render: never`，避免被 Hugo 列出）

内容：三大类 30+ 题的固定清单 + D 储备池（当主池 ≥80% 写完后启用 web_fetch 自动补题）。

**去重两层防线**：
1. 每次跑前 `ls content/学习笔记/code-agent/` + `git log --since='30 days ago'` 做**主题级**判断（不是字面匹配）
2. 池 ≥80% 写完 → `web_fetch` 抓 Anthropic / Claude Code 近期博客补 D 储备池

### 质量门禁（写完必检）

- 含 ≥1 段真实命令输出（`ps` / `ls` / `git log` / `hugo` / `launchctl` / `pmset` 等）
- 含 ≥1 段架构图（Mermaid 优先）或对比表格
- 结构完整：起点 / 原理 / 案例 / 风险 / 总结 至少 3 段
- `~/bin/hugo --minify` 通过

### front matter 模板

```yaml
---
title: "..."
date: ...
draft: true
level: beginner|intermediate|advanced
author: "min920"
author_link: "https://github.com/wm920"
categories: ["学习笔记"]
tags: [3-5 个从预设池挑]
---
```

文末必须追加：
```
> 作者：min920（GitHub @wm920） · 本文由 PiCode Agent 自动撰写并经作者审核
```

## 发布流程（每条 cron prompt 要求 agent 完整执行）

```
选题（去重）
  → 采真实命令输出（脱敏）
  → 写 .md 到 content/学习笔记/<目录>/
  → ~/bin/hugo --minify 验证（不过则重写）
  → git add/commit/push origin main
  → 飞书摘要通知（见下）
  → 任何步骤失败也发飞书说明，严禁静默退出
```

## 飞书通知规范（关键避坑点）

**必须用 `park-cli msg send --to me --text "..."`，不是 `--content`**

- `--content` 期望 JSON，传字符串会被拒（`invalid_params`）
- `--text` 接裸字符串
- agent 自用必须加 `PARK_CALLER=agent` 环境变量
- park-cli 可执行路径（macOS）：`/Users/bytedance/.local/bin/park-cli`

摘要模板（示例）：
```
📝 新草稿已就绪
《文章标题》
📁 学习笔记 / <子目录>
🏷️ level: <级别> · 类别: <A/B/C>
🔗 https://github.com/wm920/wm920.github.io/blob/main/content/<URL 编码路径>/<文件名.md>
📖 一句话摘要
✅ 发布请回复：发布《文章标题》
```

## 定时任务编排（推荐最终方案：macOS 系统 crontab）

**不要用 PiCode 内置 scheduler**——它寄生在交互式进程，没法稳定守护。picode 的 `serve` 子命令**不带 scheduler**，这是踩过的坑。

最终推荐 **macOS 系统 crontab + `picode -p` 非交互式单次模式**：

```cron
# 每条任务一次性拉起 picode -p，跑完退出，下次 fire 重来
0 9  * * *     picode -p "<FlexNOC prompt>"      --permission-mode bypass >> ~/.picode/cron-09.log 2>&1
0 13 * * *     picode -p "<协议轮换 prompt>"      --permission-mode bypass >> ~/.picode/cron-13.log 2>&1
0 15 * * 1-5   picode -p "<Code Agent prompt>"   --permission-mode bypass >> ~/.picode/cron-15.log 2>&1
0 20 * * *     picode -p "<读书/综合 prompt>"     --permission-mode bypass >> ~/.picode/cron-20.log 2>&1
30 22 * * 0,1-5 picode -p "<每日运维诊断 prompt>" --permission-mode bypass >> ~/.picode/cron-22.log 2>&1
```

Prompt 库建议存在独立 JSON/yaml 文件里，cron 行用 `jq -r` 按 id 取出（避免行太长），或者直接内嵌整段 prompt。

## macOS 环境配置（必做）

```bash
# 1. 装 Homebrew（用户自己跑，需要 sudo）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. 装 Python 3.12（避免 asyncio.Lock 在 Py3.9 下的 RuntimeError）
brew install python@3.12

# 3. venv + picode
python3.12 -m venv ~/.picode/venv312
~/.picode/venv312/bin/pip install uvicorn gunicorn fastapi pydantic python-multipart watchfiles
# picode 包从公司私有源装，或从 system Python site-packages 复制过来

# 4. pmset 唤醒（保证机器在 cron 前醒着）
sudo pmset repeat wakeorpoweron MTWRF 08:55:00

# 5. 装 jq（crontab 行里取 prompt 用）
brew install jq
```

## 每日运维诊断（**强烈建议加**）

加一条 22:30 的 cron，**只读档位 A**：

- 检查 4 条任务今天是否真跑完（git log + 磁盘 mtime + 飞书 message_id 三 AND）
- 失败时列 3 种可能性 + 证据（picode 进程 / park-cli auth / 任务本身），**不下结论**
- 飞书发极简卡片；详细报告落本地 `~/blog/.daily-report/YYYY-MM-DD.md`
- `.daily-report/` 加进 `.gitignore`
- 周日额外加周回顾（主题分布 + 选题池剩余）
- 严禁改 cron / 改 git / 补写漏掉的文章

## 踩过的坑清单（必看）

| 坑 | 症状 | 根因 | 解决 |
|---|---|---|---|
| `--content` vs `--text` | 飞书消息静默失败 | park-cli `--content` 要 JSON，`--text` 才接字符串 | 所有 prompt 改用 `--text` |
| `\|\| true` 吞错 | 通知没发，看不见 | bash 兜底太宽松 | 失败也必须显式发飞书说明 |
| `asyncio.Lock RuntimeError` | `picode serve` 在 Py3.9 下反复崩 | 3.9 同步上下文调 `asyncio.Lock()` | 升级到 Python 3.12 |
| picode serve 不带 scheduler | 守护进程跑着但任务不 fire | serve 只起 HTTP API | 改用系统 crontab |
| 机器睡眠 | cron 到点不 fire | macOS 休眠时调度器停 | `pmset repeat wakeorpoweron` |
| PiCode CronCreate durable 不落盘 | 重启后 prompt 丢 | 内部事件未同步到磁盘 | 直接编辑 `~/.picode/scheduled_tasks.json` |
| park-cli token ~24h 过期 | 飞书通知突然失败 | user token 正常 TTL | 每日运维诊断发现后弹 `park-cli auth login` 让用户点授权卡 |
| 静默字数限制 | 文章被凑/砍 | 硬字数区间 | 原则是"讲清楚优先，字数自由" |

## 给接手 Agent 的操作清单

以完全零状态接入这台机器时，**按顺序执行**：

1. **读取当前状态**
   - `ls ~/.picode/scheduled_tasks.json` 看已有 PiCode 任务（即使不再用，prompt 是资产）
   - `crontab -l` 看系统级任务
   - `launchctl list | grep picode` 看守护进程
   - `pmset -g sched` 看唤醒计划
   - `cat ~/blog/content/学习笔记/code-agent/_topics.md` 看选题池

2. **验证环境**
   - Python 3.12 venv 在 `~/.picode/venv312/bin/picode`
   - park-cli 在 `~/.local/bin/park-cli`，跑 `PARK_CALLER=agent park-cli auth ensure --format json` 看 `ready`
   - `~/bin/hugo --minify` 能在 `~/blog` 跑通

3. **理解 prompt 库**
   - 每条 cron 的 prompt 本质是一段完整 agent 指令，含：选题规则、去重策略、采数据、写作模板、Hugo 验证、git push、飞书通知、失败兜底
   - 5 条任务分别对应 FlexNOC / 协议轮换 / Code Agent / 读书综合 / 每日运维诊断

4. **触发机制选择**
   - **推荐**：系统 crontab + `picode -p`，不依赖长寿命 picode 进程
   - 不推荐：PiCode 内置 CronCreate（scheduler 寄生问题）、LaunchAgent 守护 picode serve（serve 不带 scheduler）

5. **首跑 dry-run**
   - 手动跑一次 Code Agent 任务，确认整条链路通：选题 → 写文件 → Hugo → git push → 飞书
   - 检查 `.daily-report/` 目录第一次跑时会建
   - **第一批文章可能会有质量波动**，让用户 review，迭代 prompt

## 用户偏好备忘

- 不喜欢 emoji 泛滥，但关键状态 ✅/❌/⏳/🚨 可用
- 回复要直接、中文、不要预热铺垫
- 有破坏性操作前要确认（不要自作主张 `rm` / `git reset`）
- 对 sudo / 系统配置变更保持高度警惕
- 飞书邮箱已配好，不要误以为要重新 `picode config set feishu_email`
- 当前会话已授权的 park-cli 不要反复发卡；token 过期才重发

## 验收标准（接手 Agent 自检）

完成交付后，以下都应 **Yes**：

- [ ] 连续 3 天 4 条工作日任务全部自动产出 1 篇 draft 文章
- [ ] 每篇都含真实命令输出 + 架构图/表格 + 作者署名
- [ ] 飞书每天收到 5 条消息（4 任务摘要 + 1 运维清单）
- [ ] 选题在 30 天内无主题级重复
- [ ] 任何失败都有飞书报警，不会静默
- [ ] 机器关机/睡眠重启后第二天 09:00 仍能自动 fire（pmset 唤醒验证）

---

## 这份文档的使用方式

**把整份粘贴给另一个 agent，附一句**：

> 按这份规范给我在 macOS 上搭一套每天自动写 AI Agent 博客的系统。我的博客在 `~/blog`（Hugo），GitHub @wm920，飞书邮箱已在 `picode config` 里配好。请按"给接手 Agent 的操作清单"逐步执行。
