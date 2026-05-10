---
title: "FlexNOC QoS 机制详解：如何保障关键流量的带宽与延迟"
date: 2026-05-10
draft: true
tags: ["FlexNOC", "NOC", "QoS", "服务质量", "优先级", "片上网络"]
categories: ["学习笔记"]
description: "深入解析 FlexNOC 的 QoS 机制，包括优先级模型、带宽调节、延迟保证和拥塞控制，帮助工程师为关键流量提供可靠的服务质量保障。"
---

> 在复杂 SoC 中，CPU、GPU、DMA、图像处理器等多个 master 共享 NOC 互联资源。如果没有 QoS（Quality of Service）机制，低优先级的大流量（如 DMA 数据搬运）会挤占高优先级流量（如 CPU 缓存缺失）的带宽，导致系统响应延迟不可控。本文系统解析 FlexNOC 的 QoS 体系。

## 为什么 NOC 需要 QoS

考虑以下场景：

```
同时发生的流量：
  CPU L2 Cache Miss  → DDR（延迟敏感，每 miss 阻塞 CPU）
  GPU 纹理读取        → DDR（带宽敏感，高吞吐）
  DMA 批量数据搬运    → DDR（带宽敏感，大包，低优先级）

没有 QoS 时：
  DMA 大包占满 DDR 带宽
  CPU Cache Miss 被迫等待 DMA 完成
  CPU 停顿数百纳秒 → 严重影响应用响应速度
```

FlexNOC QoS 的目标是：**在保证高优先级流量延迟的同时，让低优先级流量充分利用剩余带宽**，而不是简单地饿死低优先级请求。

---

## FlexNOC QoS 架构概览

```
Master（CPU/GPU/DMA/...）
    │
    │ 携带 QoS 标签（urgency / priority）
    ▼
┌──────────────────────────────────────────┐
│              FlexNOC 路由器               │
│  ┌────────────┐   ┌──────────────────┐  │
│  │  输入队列   │   │    仲裁器         │  │
│  │  VC0: HP  │──▶│  Strict Priority │  │
│  │  VC1: MP  │   │  或               │──▶ 输出端口
│  │  VC2: LP  │   │  Weighted RR     │  │
│  └────────────┘   └──────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │  带宽调节器（Rate Limiter / Shaper）│ │
│  └────────────────────────────────────┘ │
└──────────────────────────────────────────┘
    │
    ▼
Slave（DDR / SRAM / PCIe / ...）
```

FlexNOC QoS 由三层机制组成：
1. **优先级标签**：每个事务携带 urgency 字段
2. **虚通道隔离**：不同优先级使用独立队列
3. **仲裁策略**：决定多个队列之间的服务顺序

---

## 一、QoS 优先级标签

### AXI QoS 信号

FlexNOC 基于 AXI 协议的 `AWQOS`/`ARQOS` 信号携带 4-bit 优先级值（0~15，15 最高）：

```
AXI Master 发出事务时：
  ARADDR = 0x8000_0000   (目标地址)
  ARQOS  = 4'b1111       (最高优先级，适合 CPU)
  ARQOS  = 4'b0100       (中等优先级，适合 GPU)
  ARQOS  = 4'b0000       (最低优先级，适合后台 DMA)
```

### FlexNOC Urgency 扩展

FlexNOC 在 AXI QoS 基础上扩展了 **Urgency** 概念，支持更细粒度的优先级控制：

| Urgency 级别 | 含义 | 典型用途 |
|-------------|------|---------|
| Critical (3) | 立即服务，不可被抢占 | 实时控制、安全关键路径 |
| High (2) | 高优先级，可短暂等待 | CPU 指令 Fetch、L2 Miss |
| Normal (1) | 正常服务 | GPU 计算访问 |
| Low (0) | 尽力而为 | DMA 批量搬运、预取 |

---

## 二、虚通道（VC）隔离

虚通道是 FlexNOC QoS 的物理基础——不同优先级的流量在物理链路上共享带宽，但使用独立的缓冲队列：

```
物理链路（256-bit，@1GHz）
     │
     ├── VC0（Critical）：独立 FIFO，深度 4
     ├── VC1（High）    ：独立 FIFO，深度 8
     ├── VC2（Normal）  ：独立 FIFO，深度 16
     └── VC3（Low）     ：独立 FIFO，深度 32
```

**VC 隔离的关键价值：Head-of-Line（HoL）阻塞消除**

```
没有 VC 时：
  队列：[DMA大包][DMA大包][DMA大包][CPU Miss]
        ↑ DMA 占据队头，CPU Miss 被迫等待

有 VC 时：
  VC_High：[CPU Miss]           ← 立即服务
  VC_Low ：[DMA大包][DMA大包]   ← 等待 VC_High 空闲后服务
```

### FlexNOC VC 配置示例

```
# FlexNOC 配置参数（Arteris FlexNOC 设计工具语法示意）
socket_config {
  name: "cpu_master"
  vc_count: 4
  vc_mapping: {
    qos_range [12:15] -> VC0  # Critical
    qos_range [8:11]  -> VC1  # High
    qos_range [4:7]   -> VC2  # Normal
    qos_range [0:3]   -> VC3  # Low
  }
}
```

---

## 三、仲裁策略

当多个 VC 同时有数据待发送时，仲裁器决定服务顺序。FlexNOC 支持以下仲裁模式：

### 3.1 严格优先级（Strict Priority）

高优先级 VC 始终优先服务，低优先级只有在高优先级 VC 为空时才获得服务：

```
仲裁决策：
  if VC0 not empty → 服务 VC0
  else if VC1 not empty → 服务 VC1
  else if VC2 not empty → 服务 VC2
  else → 服务 VC3
```

**优点**：关键流量延迟最低
**缺点**：低优先级流量可能**饥饿（Starvation）**

### 3.2 加权轮询（Weighted Round Robin, WRR）

每个 VC 分配权重，按权重比例获得服务机会：

```
权重配置：VC0:VC1:VC2:VC3 = 8:4:2:1

仲裁轮次示例（共 15 个 slot）：
VC0 获得 8 次服务机会
VC1 获得 4 次服务机会
VC2 获得 2 次服务机会
VC3 获得 1 次服务机会
```

**优点**：防止低优先级饥饿，带宽分配可预测
**缺点**：不能保证 Critical 流量的绝对最低延迟

### 3.3 混合仲裁（Hybrid：SP + WRR）

FlexNOC 推荐的生产配置：

```
Critical VC（VC0）：严格优先级，绝对抢占
High/Normal/Low VC（VC1~VC3）：WRR，防止饥饿

伪代码：
  if VC0 not empty:
      serve VC0  // Critical 无条件优先
  else:
      serve VC1/2/3 by WRR weights  // 其余按权重轮询
```

---

## 四、带宽调节（Rate Limiting）

除了优先级仲裁，FlexNOC 还支持对特定 master 进行带宽上限限制，防止单个 master 独占总线：

```
Rate Limiter 配置示例：
  DMA master：最大带宽 = 总带宽的 40%
  GPU master：最大带宽 = 总带宽的 50%
  CPU master：不限制（保证突发响应）

实现原理：令牌桶（Token Bucket）算法
  桶容量 = burst_size（允许的突发大小）
  填充速率 = 目标带宽
  每发送一个 flit 消耗一个 token
  token 不足时，事务进入等待队列
```

---

## 五、延迟保证机制

对于延迟敏感的流量（如 CPU），FlexNOC 提供**最坏情况延迟界限（Worst-Case Latency Bound）**分析能力：

```
延迟上界分析输入：
  - 每条路径上的链路数（跳数）
  - 每个路由器的仲裁延迟（由权重和竞争情况决定）
  - 虚通道缓冲深度

输出：
  对于 Critical 流量，在最坏竞争情况下，
  从 Master 到 Slave 的最大延迟 = X ns

FlexNOC 配置工具会自动计算并报告此值，
工程师可据此验证是否满足实时性要求。
```

---

## QoS 配置实战：典型 SoC 场景

```
SoC 组成：
  - 4核 CPU（高优先级缓存访问）
  - GPU（中优先级纹理/显存访问）
  - 视频编解码器（需要稳定带宽，延迟不敏感）
  - 后台 DMA（低优先级）

推荐 QoS 配置：
┌──────────────────┬────────┬────────┬──────────────┐
│ Master           │ ARQOS  │ VC     │ 带宽上限     │
├──────────────────┼────────┼────────┼──────────────┤
│ CPU L2 Miss      │ 15     │ VC0    │ 无限制       │
│ CPU Prefetch     │ 8      │ VC1    │ 20%          │
│ GPU Compute      │ 10     │ VC1    │ 50%          │
│ Video Codec      │ 6      │ VC2    │ 30% (保底)   │
│ DMA Background   │ 0      │ VC3    │ 40%          │
└──────────────────┴────────┴────────┴──────────────┘
```

---

## 小结

FlexNOC QoS 是一套分层机制：**优先级标签**标识流量重要性，**虚通道**提供物理隔离防止 HoL 阻塞，**仲裁策略**在保证高优先级延迟的同时防止低优先级饥饿，**带宽限速**防止单个 master 独占资源。工程实践中，推荐使用混合仲裁（Critical 严格优先 + 其余 WRR），并结合 FlexNOC 配置工具的延迟分析功能验证关键路径满足实时性要求。

*本文为草稿，待审阅发布。*
