---
title: "从 TCP 到 RDMA：入门与核心概念"
date: 2026-05-10
draft: true
tags: ["RDMA", "InfiniBand", "RoCE", "Verbs", "网络", "高性能"]
categories: ["学习笔记"]
description: "对比传统 TCP/IP 内核协议栈开销，介绍 RDMA 的三种传输类型、核心操作（Send/Recv/Read/Write）、以及常用 Verbs API 调用示例。"
---

## 为什么需要 RDMA

在数据中心和高性能计算场景中，传统 TCP/IP 协议栈的主要瓶颈在**内核态拷贝**和**上下文切换**：

```
  应用发送数据
      ↓
  用户态缓存拷贝到内核态
      ↓
  TCP/UDP 处理
      ↓
  IP 分片、路由
      ↓
  网卡队列调度
      ↓
  发送到网络
      ↓
  接收端反向流程
      ↓
  内核态拷贝到用户态
```

每次收发都涉及用户态 ↔ 内核态切换，且多次内存拷贝，CPU 开销巨大，延迟通常在几十微秒到毫秒级别。

RDMA（Remote Direct Memory Access）直接绕过内核，由硬件（网卡）完成数据收发，实现：
- **零拷贝**：收发直接在用户态缓存和远程内存之间移动
- **内核旁路**：不经过内核协议栈
- **CPU 占用极低**：收发几乎不消耗 CPU

典型 RDMA 单跳往返延迟仅 **1~5μs**，且带宽接近物理线速。

---

## RDMA 的三种网络传输类型

| 名称 | 简称 | 网络类型 | 兼容性 | 适用场景 |
|------|------|----------|--------|----------|
| InfiniBand | IB | 专用 IB 网络 | 最高 | HPC、超算 |
| RoCEv1 | — | 以太网 | 仅同子网 | 同机架集群 |
| RoCEv2 | — | 以太网 | 跨子网路由 | 通用数据中心 |
| iWARP | — | 以太网 TCP | TCP 兼容 | 广域网、对丢包敏感场景 |

RoCE 是目前主流，底层还是以太网，上层协议替换成 RDMA 传输层。

---

## 核心概念：QP、MR、CQ

RDMA 通信模型和 Socket 完全不同，核心是三个对象：

```
  应用端        硬件（网卡）
    ↓               ↓
  MR（内存注册） ← 锁定物理内存，允许网卡直接访问
    ↓
  QP（队列对） ← 收发队列，每对 QP 一对一连接远程 QP
    ↓
  工作请求（WR）
    ↓
  工作完成（WC）
    ↓
  CQ（完成队列） ← 轮询或事件通知
```

### 1. 内存注册（MR）
网卡要访问用户态内存，必须提前**注册**：
- 锁定物理页（不能被 swap 换出）
- 网卡建立虚拟/物理地址映射
- 返回访问权限（本地读/本地写/远程读/远程写/原子操作）

```c
struct ibv_mr *mr;
mr = ibv_reg_mr(pd, buf, size,
                IBV_ACCESS_LOCAL_WRITE |
                IBV_ACCESS_REMOTE_READ |
                IBV_ACCESS_REMOTE_WRITE);
```

> 使用完要 `ibv_dereg_mr(mr)` 释放注册资源。

### 2. 队列对（QP）
QP 对应两个队列：
- **SQ（发送队列）**：发请求（Send/Write/Read/原子操作）
- **RQ（接收队列）**：收请求（Recv/对端发的 Send）

每对 QP 和对端 QP 是**一对一**关系，初始化时要交换地址和 QPN（QP 编号）。

### 3. 完成队列（CQ）
硬件处理完 WR 后，产生 WC 放在 CQ 里：
- 状态（成功/失败）
- 字节数
- QP 编号
- WR ID（标识是哪一个请求完成）

---

## 核心操作

### 基本四类操作

| 操作 | 英文 | 方向 | 特点 |
|------|------|------|------|
| 发送 | Send | 主动 → 被动 | 双端都要发 WR：对端先 `Post Recv` |
| 接收 | Recv | 被动 ← 主动 | 准备接收缓冲区，硬件填入数据 |
| 远程写 | Write | 本地 → 远端 | 单边：本地只需要知道远端地址和 rkey，对端不知道数据到了 |
| 远程读 | Read | 本地 ← 远端 | 单边：本地把远端数据拉过来，对端不知道 |

**单边操作 vs 双边操作**
- **双边（Send/Recv）**：两端都要发 WR
- **单边（Read/Write）**：本地知道地址和密钥直接读写，对端不感知，适合大块数据搬运

---

## 简单 Verbs API 调用示例

### 初始化流程

```c
// 1. 获取设备列表
struct ibv_device **dev_list;
dev_list = ibv_get_device_list(&num_devices);

// 2. 打开设备
struct ibv_context *ctx;
ctx = ibv_open_device(dev_list[0]);

// 3. 分配保护域（PD）
struct ibv_pd *pd;
pd = ibv_alloc_pd(ctx);

// 4. 创建 CQ
struct ibv_cq *cq;
cq = ibv_create_cq(ctx, 100, NULL, NULL, 0);

// 5. 创建 QP
struct ibv_qp_init_attr qp_attr = {
  .send_cq = cq,
  .recv_cq = cq,
  .cap = {
    .max_send_wr = 100,
    .max_recv_wr = 100,
    .max_send_sge = 1,
    .max_recv_sge = 1,
  },
  .qp_type = IBV_QPT_RC, // RC（可靠连接）最常用
};
struct ibv_qp *qp;
qp = ibv_create_qp(pd, &qp_attr);
```

### 发送操作（Send）

```c
struct ibv_sge sge = {
  .addr = (uintptr_t)buf,
  .length = size,
  .lkey = mr->lkey,
};
struct ibv_send_wr wr = {
  .wr_id = 123,
  .sg_list = &sge,
  .num_sge = 1,
  .opcode = IBV_WR_SEND,
  .send_flags = IBV_SEND_SIGNALED,
};
struct ibv_send_wr *bad_wr;
ibv_post_send(qp, &wr, &bad_wr);
```

### 轮询 CQ 等待完成

```c
struct ibv_wc wc;
while (ibv_poll_cq(cq, 1, &wc) == 0) {
  // 可加一点短暂 sleep 避免空转
}
if (wc.status != IBV_WC_SUCCESS) {
  fprintf(stderr, "WC failed: %s\n", ibv_wc_status_str(wc.status));
}
```

---

## 总结
RDMA 入门核心是理解：
1. 为什么要 RDMA（内核开销、延迟对比）
2. RDMA 对象模型（QP、MR、CQ）
3. 四类操作（Send/Recv/Write/Read）
4. 基本 Verbs API 调用流程
