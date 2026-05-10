---
title: "《计算机体系结构：量化研究方法》读书笔记——存储层次结构"
date: 2026-05-11
draft: true
tags: ["读书笔记", "计算机体系结构", "存储层次", "Cache", "DDR", "性能优化"]
categories: ["读书笔记"]
description: "Hennessy & Patterson 经典著作第二章精读笔记，聚焦存储层次结构的设计哲学、Cache 工作原理与现代 SoC 中的实际挑战。"
---

> 「Memory is the bottleneck. It always has been, and it probably always will be.」
> —— 帕特森，《计算机体系结构：量化研究方法》

## 为什么反复读这本书

《计算机体系结构：量化研究方法》（*Computer Architecture: A Quantitative Approach*，Hennessy & Patterson）是体系结构领域的圣经级教材。我第一次读它是在校期间，囫囵吞枣；工作之后接触 SoC 设计，才发现书里每一个细节都对应着实际工程中踩过的坑。这篇笔记聚焦第二章——**存储层次结构**，这是我认为全书工程价值最高的一章。

---

## 核心观点一：局部性原理是一切的基础

整个存储层次的设计哲学，建立在一个朴素的经验规律上：

> *Temporal locality*: a recently accessed item is likely to be accessed again soon.
> *Spatial locality*: items near a recently accessed item are likely to be accessed soon.

**时间局部性**和**空间局部性**——这两条原理支撑着从 L1 Cache 到虚拟内存的全部设计决策。

书中有一个让我印象深刻的数据：在典型程序中，90% 的执行时间花在 10% 的代码上。这正是局部性原理的直接体现，也是为什么一个容量远小于内存的 Cache 能带来接近全命中的性能。

**我的思考**：在 SoC 设计中，NOC（片上网络）的流量模型同样依赖局部性假设——CPU 的访存流量高度局部化，而 DMA 的批量搬运却完全破坏空间局部性，这正是两者在 QoS 配置上需要区别对待的根本原因。

---

## 核心观点二：AMAT 公式——量化 Cache 性能的利器

书中给出了**平均内存访问时间（AMAT，Average Memory Access Time）**的经典公式：

```
AMAT = Hit Time + Miss Rate × Miss Penalty
```

看起来简单，但展开之后会发现层层嵌套：

```
AMAT_L1 = Hit_L1 + MissRate_L1 × (Hit_L2 + MissRate_L2 × MissTime_DRAM)
```

书中用这个公式系统分析了三类 Cache miss：

| Miss 类型 | 英文 | 成因 | 可优化性 |
|-----------|------|------|---------|
| 强制缺失 | Compulsory Miss | 首次访问，冷启动 | 预取可缓解 |
| 容量缺失 | Capacity Miss | Cache 装不下工作集 | 增大 Cache |
| 冲突缺失 | Conflict Miss | 映射冲突，组相联度不足 | 提升相联度 |

> The three C's of cache misses provide a framework for thinking about what can be improved and at what cost.

**关键摘录**：书中强调，在现代处理器中，**Miss Penalty 是主要矛盾，而不是 Miss Rate**。即使把 Miss Rate 从 2% 降到 1%，如果 DRAM 延迟从 100ns 增加到 200ns，AMAT 反而变差。这在 DDR5 时代尤为值得警惕——带宽大幅提升，但延迟的改善幅度远不如带宽。

---

## 核心观点三：写策略的工程取舍

Cache 的写操作有两种基本策略，书中对此有非常清晰的对比：

**写直达（Write-Through）**
```
写命中 → 同时更新 Cache 和内存
优点：实现简单，Cache 与内存始终一致
缺点：每次写都产生内存流量，写带宽压力大
```

**写回（Write-Back）**
```
写命中 → 只更新 Cache，标记 Dirty bit
写缺失时 → 将 Dirty 行替换到内存后再填充
优点：写流量大幅减少，突发写性能好
缺点：Cache 与内存可能不一致，多核一致性更复杂
```

> In a write-back cache, the cache absorbs many writes, reducing the memory bandwidth required.

**我的思考**：现代 SoC 多核处理器几乎清一色使用写回策略，但这给 Cache 一致性协议（如 MESI）带来了极大的复杂度。在做 NOC 互联设计时，Snoop 流量（Cache 一致性探测请求）往往是一个容易被忽视的带宽消耗来源，而它的根源正是写回策略带来的数据不一致窗口。

---

## 核心观点四：预取的双刃剑

书中专门讨论了硬件预取（Hardware Prefetch）：

> Prefetching can reduce miss rates dramatically, but if done poorly, it can pollute the cache and increase bandwidth usage without helping performance.

预取的收益很直观：把将来会用到的数据提前搬进 Cache，把 Miss Penalty 隐藏在计算背后。但书中特别指出两个反效果：

1. **Cache 污染**：预取了不会被用到的数据，驱逐了有用的行
2. **带宽浪费**：无效预取占用内存带宽，反而让真正的 miss 等待更久

**量化结论**：书中引用实验数据，激进的顺序预取在流式访问场景下可以把 DRAM 带宽利用率提升到 85% 以上；但在随机访问场景下，同样的预取策略会使性能下降 20~30%。

这和我实际观察到的 NVMe SSD 行为高度吻合——顺序读的预读效果显著，而随机小 IO 场景下预读反而是负担。

---

## 让我重新理解的一个细节：Non-Uniform Cache Access

书中在讨论多核架构时提到了 NUCA（Non-Uniform Cache Access）：

> As the number of cores increases, the time to access a shared last-level cache becomes non-uniform depending on which bank of the cache is accessed.

在小核数时代，LLC（Last Level Cache）的访问延迟基本均匀。但当核数超过 16 甚至 64 时，LLC 物理上不得不分布在芯片各处，不同核访问不同 Bank 的延迟开始出现差异——就像 NUMA 内存访问一样。

这个问题在今天的 Chiplet 多芯片封装时代被进一步放大。书中 2017 年版本对此的讨论还比较简略，但它埋下的思路——**存储层次的非均匀性是规模扩展的必然代价**——在今天看来极具预见性。

---

## 数字速查：不同层次的典型延迟

书中附录给出了一张延迟对比表，我整理成更直观的形式：

```
存储层次          典型延迟        典型容量
─────────────────────────────────────────
寄存器            < 1 cycle       < 1 KB
L1 Cache          4~5 cycles      32~64 KB
L2 Cache          12~15 cycles    256 KB~1 MB
L3 Cache          40~50 cycles    8~64 MB
DDR5 DRAM         ~70 ns          GB 级
NVMe SSD          ~100 μs         TB 级
HDD               ~10 ms          TB 级
─────────────────────────────────────────
相邻层级延迟差：约 3~10×
```

这张表贴在工位上已经好几年了。每次有人问「为什么内存访问这么慢」，我就指着它说：你觉得慢是因为你在和寄存器比；要是和磁盘比，内存已经是光速了。

---

## 读完的困惑与延伸问题

1. **CXL 如何重塑存储层次？** 书中的层次结构假设 CPU 与内存通过固定总线连接。CXL（Compute Express Link）允许把远端内存、加速器内存统一纳入地址空间——这是在 DDR 和 NVMe 之间插入了新的一层，还是彻底打破了层次结构的假设？

2. **HBM 的位置** 高带宽内存（HBM）物理上封装在 GPU/AI 芯片旁边，延迟接近 DRAM 的一半，带宽高出 5~10 倍——它算 L4 Cache，还是只是更快的 DRAM？书中对此没有明确讨论，但这个问题在 AI 芯片设计中越来越重要。

3. **多 Die 封装下的 Cache 一致性** Chiplet 架构中，不同 Die 上的 L3 Cache 之间的一致性维护代价是多少？这是 2006 年初版时完全没有的工程问题。

---

## 小结

这一章最大的价值不是具体的 Cache 设计参数，而是**量化思维方式**：用 AMAT 公式把模糊的「Cache 性能」拆解为可测量、可优化的三个变量，再在三个变量之间做工程取舍。这种分析框架在今天面对 DDR5、HBM、CXL 等新技术时依然适用。

下一次打算读第三章（指令级并行），ILP 和 OOO 执行的内容在理解现代 CPU 微架构时同样基础而关键。
