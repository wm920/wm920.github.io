---
title: "为什么传统 EDA 工作流难以 agent 化"
date: 2026-05-13T22:00:00+08:00
draft: false
tags: ["code-agent", "EDA", "chip-design", "meta", "automation"]
categories: ["学习笔记", "code-agent"]
description: "软件圈 agent 风风火火，EDA 圈却静得出奇。不是 EDA 工程师不想自动化，是五堵现实的墙——闭源工具、license 瓶颈、仿真耗时、隐式知识、NFS 数据——让 agent 进不来。本文逐条拆解并给处方。"
---

## 一个反差

2025 上半年开始，软件圈的 agent 已经铺天盖地——Claude Code、Cursor、Aider、各家公司自己的 code agent 每天都在写 PR。

同一时期的 EDA（芯片设计自动化）世界几乎安静得诡异：大家都在谈"AI 辅助设计"，但**真正把 agent 接进 RTL / DV / PD / DFT 日常 flow 并 24 小时跑着**的公司，能举出来的屈指可数。

这不是 EDA 工程师守旧。恰恰相反，EDA 工程师比软件工程师更爱用脚本、更爱自动化——Perl / Tcl / Makefile 的密度是 EDA 工作流的底色。问题在于：**软件圈 agent 那套循环（读代码→跑命令→看输出→修改→验证）在 EDA 工作流里有五个结构性障碍**，不是 prompt 调一调能绕过去的。

本文把这五堵墙逐条拆开，每条给一个最现实的处方。

## 障碍 1：EDA 工具链闭源二进制，难观测

写个 Python 脚本 bug 了，agent 可以 `print`，可以 `pdb`，可以读 traceback。

跑 Synopsys DC / Cadence Innovus / VCS 挂了，输出通常是一坨几十 MB 的 log，内嵌了你看不懂的内部错误码、license debug 信息、临时路径。工具本身是 C++ 编译的闭源二进制，agent **没办法 step 进去**，也**没办法在其中插桩**。

### 这事有多严重

拿开源 RTL 仓库体量打个底：OpenTitan 这种相对"小"的项目：

```bash
$ du -sh ~/.tmp/agent-blog-cache/opentitan
305M    /Users/bytedance/.tmp/agent-blog-cache/opentitan

$ find ~/.tmp/agent-blog-cache/opentitan -name "*.sv" | wc -l
3946
```

**单仓库 305 MB、3946 个 SystemVerilog 文件**。一次完整 synthesis 的 log 输出和中间产物能再乘以 10 倍。agent 连读完一次 run 都吃力，更别说理解里面的工具私有术语（什么叫 `SDG-175` warning？什么叫 `UPF-023`？）。

### 处方

1. **不要让 agent 去"懂工具"，让 agent 去"读 report"**。综合/PnR 的 `*.rpt` 文件有标准格式，agent 解析其中的 setup/hold violation 表和 utilization 比解析 log 容易 10 倍。
2. **给 agent 一个工具专家层**：对每个 EDA 工具写一个薄 wrapper，把最常见的 warning / error 归类成"可操作项"+"纯 info"两类，agent 只消费可操作项。
3. **错误码知识库**。和内部 EDA 支持团队合作，把最常见 50 条 warning 的含义 / 根因 / 处方写成 markdown，agent RAG 一下比瞎猜强。

## 障碍 2：License 单点瓶颈

软件世界的 agent 可以并发：启 10 个 sandbox、跑 10 份 pytest，互不干扰。

EDA 世界**license 本身就是稀缺资源**。一个 site 可能只有 8 份 VCS feature、2 份 Primetime、1 份 Conformal LEC。这些 license 是工程师共享的，agent 并发申请就是**和同事抢饭**。更糟的是 license 释放不及时会触发工具内部死锁，一个工程师出差忘了退 license，全组等三天。

### 处方

1. **硬配额**。agent 每天申请的 license-hours 有上限，超过报警不让跑。
2. **错峰调度**。人类白天上班的高峰时段 agent 禁跑，夜间和周末跑 regression。这也符合 EDA 团队惯例。
3. **优先本地/开源工具**。能用 verilator 做的 smoke 测试绝不动 VCS；能用 yosys + openroad 做的初步探索绝不动商业 synth flow。
4. **监控 license 占用**：agent 每个任务开始前读一次 `lmstat`，没余量就排队不要抢。

## 障碍 3：仿真耗时破坏反馈环

软件圈 agent 循环的本质是**"快速迭代"**：改一行 → 5 秒看到结果 → 再改一行。

EDA 的反馈环是什么量级？

| 阶段 | 单次典型时长 |
|---|---|
| Lint（Verilator / Spyglass） | 1-10 分钟 |
| Unit 级仿真（UVM 单测） | 10 分钟-1 小时 |
| Block regression（几十条 case） | 4-8 小时 |
| SoC 集成 regression | 8-24 小时 |
| 综合（DC / Genus） | 2-8 小时 |
| PnR 一轮 | 6-24 小时 |
| Gate-level regression | 1-3 天 |
| Silicon bring-up | 几周-几月 |

一次反馈 8 小时意味着**每天 agent 最多迭代 3 次**。对比软件 agent 一天可以试 100 次——**采样率差了 30 倍**，学习曲线和纠错能力完全不是一个档次。

Makefile 密度也能侧面印证工作流有多 heavy。随手看一下 CVA6：

```bash
$ find ~/.tmp/agent-blog-cache/cva6 -name "Makefile" | head -5
/Users/bytedance/.tmp/agent-blog-cache/cva6/pd/synth/Makefile
/Users/bytedance/.tmp/agent-blog-cache/cva6/core/pmp/Makefile
/Users/bytedance/.tmp/agent-blog-cache/cva6/verif/tests/custom/debug_test/bsp/Makefile
/Users/bytedance/.tmp/agent-blog-cache/cva6/verif/tb/core/Makefile
/Users/bytedance/.tmp/agent-blog-cache/cva6/verif/tb/core/bootrom/Makefile
```

synth、PMP、各层 verif、bootrom ——每一层都有独立的 Makefile 封装上千条子任务。每一层都要 agent 会读 Make 输出，并且都是分钟-小时级。

### 处方

1. **收缩反馈环**。能用 smoke test（5 分钟）验证的事情绝对不跑完整 regression。agent 先跑最小复现，确认方向对了再扩。
2. **并行 worktree**。git worktree + 分布式作业调度（LSF / SLURM），把 1×8h 变成 4×2h。
3. **善用 cache**：ccache、sim binary cache、regression result cache。增量 regression 只跑受影响的 case。
4. **async agent**：agent 不阻塞式等结果，改成 event-driven——任务提交、结果回来再唤醒继续。这才是 EDA 里的 agent 该有的形态。

## 障碍 4：工程师知识隐式，不在代码里

软件项目的知识大多写在代码、test、README、git commit message 里，agent 读得到。

EDA 不一样。一个典型场景：

> "这个 `clkgate_cell` 必须用 `CKGATE_X4` 变体，不能用 `X2`。"
> 
> "为什么？"
> 
> "因为去年流片的时候 X2 在 corner SS 125C 跑出过 setup violation。"
> 
> "这件事哪里写着？"
> 
> "……老张脑子里，他上个月离职了。"

**EDA 工程师 20% 的工作知识**都是这种形态——某个配置的 magic number、某个 corner 的 waiver、某个 IP vendor 的 errata——**从来不在 git 里**。agent 读完整个仓库也学不会。

### 处方

1. **显式化是长期工程**。每次老师傅解一个问题，要求同时写一条带上下文的 ADR（Architecture Decision Record）到仓库。这件事本身可以 agent 辅助——采访式生成草稿。
2. **检索隐式知识的替代来源**：飞书群聊、邮件存档、评审会议纪要。把这些喂给 agent 做 RAG 比硬学 RTL 有效。
3. **让 agent 先做"我不知道"**。比起自己胡编，不如设计 agent 遇到隐式配置（magic number / 特殊 waiver）就停下来问人——把不确定性暴露给工程师，而不是藏着。
4. **承认上限**。某些隐式知识 agent 这辈子都学不会，承认这一点、合理划定 agent 适用边界，比假装全能更可持续。

## 障碍 5：数据在 NFS / LSF，不在 git

软件项目的整个 state 都在 git 里——clone 仓库就拿到完整上下文。

EDA 不是。一个典型的 block：

- RTL 和 DV 代码在 git（~1 GB 量级）
- 综合 / PnR 的 flow 脚本在 git 或单独的 PD 仓库
- **中间产物、netlist、SDF、SPEF、lib 都在 NFS `/project/xxx/design/block_a/runs/2026-05-13/`**（几十 GB 到 TB）
- 仿真 wave / FSDB 在另一块 NFS
- LSF / SLURM 作业记录在集群调度器数据库
- Tapeout 版本的 lock 文件在 PDM / PLM 系统

agent 要做任何分析都要**同时访问 git + NFS + LSF + 可能的 Web PDM**，每一层都有各自的权限、挂载、超时、cache 问题。软件圈那种 `git clone && agent run` 的闭环在 EDA 里根本不存在。

### 处方

1. **把"访问层"抽象出来**。给 agent 一个 `artifact_store` 接口，具体后端可以是 NFS / S3 / 内部对象存储，agent 不关心细节。路径约定 + 稳定 API 是基础设施，做一次受益所有 agent。
2. **元数据先行**。agent 大部分时候其实不需要读几十 GB 的 FSDB，只需要读"某次 run 的 summary"。把每次 run 的 summary JSON 化，index 到一个轻量 DB，让 agent 先查再决定是否加载大文件。
3. **Git-native 子集**。能塞进 git 的 metadata（脚本、配置、小 report）优先塞进去。不要盲目全塞——VCS 产物进 git 会把仓库撑爆。拿捏在"对 agent 有用 + 体积 <1MB"这条线。
4. **就近计算**。agent 运行环境就放在能直接挂 NFS 的跳板机 / 集群 login node 上，别跑在某个云 VM 上再绕 VPN 回来读数据，延迟和权限问题会烦到想砸电脑。

## 五堵墙速查表

| # | 障碍 | 表现 | 根因 | 优先处方 |
|---|---|---|---|---|
| 1 | 闭源工具难观测 | Log 巨大、错误码晦涩 | EDA vendor 商业模式 | 解析 report 而非 log，建工具 wrapper 层 |
| 2 | License 单点瓶颈 | agent 跑着抢了同事饭 | feature-based pricing | 硬配额 + 夜间调度 + 优先开源工具 |
| 3 | 仿真耗时破坏反馈环 | 一天迭代 3 次 | 硬件仿真的 inherent cost | Smoke-first、并行 worktree、async agent |
| 4 | 工程师知识隐式 | Magic number 来自脑子 | 行业 apprenticeship 文化 | ADR 显式化 + "我不知道"设计 + 群聊 RAG |
| 5 | 数据在 NFS 不在 git | `git clone` 拿不全上下文 | 历史基础设施分层 | 统一 artifact 接口 + 元数据索引 + 就近计算 |

## 那 EDA 到底有没有 agent 化甜区

五堵墙是结构性的，但不是无解。把它们反过来看就是甜区：

1. **凡是只需要读 text report 的工作**都好做——lint 整理、综合结果 summary、timing 报告的 top violator 列举。
2. **凡是不消耗昂贵 license 的工作**都好做——Verilator / Yosys / OpenROAD 这一栈开源工具可以给 agent 无限跑。
3. **凡是反馈环 < 10 分钟的工作**都好做——Lint、编译检查、pre-commit hook、文档同步、cherry-pick。
4. **凡是知识可以显式化的工作**都好做——规范检查、命名一致性、register 定义 vs SystemRDL。
5. **凡是数据能进 git 的工作**都好做——RTL 本身、DV 脚本、Make flow、ADR 管理。

把这五类交集起来，你会发现：**EDA agent 的真实甜区不是"替代 EDA 工程师"，是"在非昂贵、可观测、快速反馈的外围流程上做胶水"**。这和软件 agent 那种"直接写功能"的玩法完全不同，但并不比后者小众。

## 给团队的建议

如果你正在芯片团队里试推 agent：

1. **不要一上来就攻 PnR / 综合**。那是五堵墙最厚的地方。先从 RTL lint 整理、commit message 规范、文档同步这种"墙外工作"做起，建立信任度。
2. **把 license 和集群配额写进 agent 的 system prompt**。让 agent 自觉知道这些是稀缺资源，主动节制。
3. **把"不确定就问人"设为默认行为**，而不是"不确定就猜"。EDA 工程师比软件工程师对"幻觉"的容忍度低得多——一次错误的 CKGATE 选择可能对应几百万的重 tape。
4. **从 agent 用户出发**，不从 agent 能力出发。问 EDA 工程师"你每天最烦做哪件机械工作"，让 agent 做那件事，比炫技十次都有效。

## 收尾

软件圈 agent 的成功建立在**"可观测、便宜、快、文本化"**这四个隐含假设上。EDA 世界在这四个维度每一条都打折扣，所以 agent 渗透进来会比软件慢得多——这不是 EDA 落后，是 EDA 的问题域本身更难。

但"慢"不等于"不会"。认清墙的位置，绕过去，在墙外的辽阔空间里扎实做事，比硬撞墙实在。

---

> 作者：min920（GitHub @wm920） · 本文由 PiCode Agent 自动撰写并经作者审核
