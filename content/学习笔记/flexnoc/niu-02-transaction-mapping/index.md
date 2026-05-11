---
title: "FlexNOC NIU 事务处理（02）：事务映射与拆分 [中英对照]"
date: 2026-05-10
draft: true
series: ["FlexNOC NIU Transaction Handling"]
series_order: 2
tags: ["FlexNOC", "NIU", "片上网络", "事务处理", "burst splitting", "AXI", "AHB", "OCP"]
categories: ["学习笔记"]
source: "Arteris IP - FlexNOC NIU Transaction Handling Technical Reference (FlexNoC 4.5.1, 2021-02-27)"
description: "FlexNOC NIU 事务处理系列第二篇，详解事务映射与拆分的七步流程：窄突发打包、精确突发拆分、MRMD 转 SRMD、地址解码与基于目标的拆分、回绕突发对齐、字对齐，以及突发拆分与 SRMD→MRMD 转换。含字节序处理说明。"
---

> **说明**：本系列基于 Arteris IP《FlexNoC NIU Transaction Handling Technical Reference》（FlexNoC 4.5.1，2021年2月27日版）逐章翻译整理，采用中英双语对照排版，原文以引用块呈现，译文紧随其后。信号名、参数名、寄存器字段名保留英文原样。

---

## Chapter Two — NIU Transaction-handling Operations（第二章：NIU 事务处理操作）

> NIU transaction handling operations are characterized by:
>
> - Socket protocol, in particular for third-party protocol specific NIUs.
> - NoC NIU type: initiator, target, specific, or generic.
> - The type of transaction emitted by the master IP, that is, transaction attributes as specified by protocol standards.
> - Other NoC parameters that configure end-to-end behavior between the initiator and the target.

NIU 事务处理操作由以下因素决定：

- Socket 协议，尤其对第三方协议专用 NIU 而言。
- NoC NIU 类型：initiator（发起方）、target（目标方）、specific（专用）或 generic（通用）。
- Master IP 发出的事务类型，即协议标准规定的事务属性。
- 配置 initiator 与 target 之间端到端行为的其他 NoC 参数。

> **Note:** Address decoding and address map setup is covered in related documentation.

**注意：** 地址解码和地址映射配置详见相关文档。

本章涵盖以下主题：

| 小节 | 内容 |
|---|---|
| Transaction mapping and splitting | 事务映射与拆分（本篇） |
| Endianness handling | 字节序处理（本篇末） |
| Request and response user info transport | 请求/响应用户信息传输 |
| Memory types, posted writes and early write responses | 内存类型、posted 写与提前写响应 |
| Partial accesses and byte enables | 部分访问与字节使能 |
| Special bursts | 特殊突发 |
| Atomic sequences and exclusive accesses | 原子序列与独占访问 |
| Read-only and write-only NIUs | 只读/只写 NIU |
| Configuring multi-port NIUs | 多端口 NIU 配置 |
| NIUs with multiple generic interfaces | 具有多个通用接口的 NIU |
| Service network target NIU | 服务网络 target NIU |
| Error generation | 错误生成 |

---

## Transaction mapping and splitting（事务映射与拆分）

> Transaction mapping and splitting is the process by which initiator transactions are mapped into one or several transactions first to the fabric, then to targets or to errors. This section describes the process focusing on incrementing and wrapping transactions, which are the two bursts types that the generic interface and fabric natively support. Some exceptions to this process may occur because of transaction attributes (byte enables, memory types) and these exceptions are documented in a separate section of this chapter.

事务映射与拆分（transaction mapping and splitting）是一个将 initiator 事务映射为一个或多个事务的过程，这些事务先发往 fabric，再发往 target 或生成错误响应。本节重点描述递增突发（incrementing burst）和回绕突发（wrapping burst）的处理过程——这是通用接口和 fabric 原生支持的两种突发类型。由于事务属性（字节使能、内存类型）的影响，该过程存在若干例外情况，将在本章单独小节中说明。

> Request mapping happens as a series of steps involving the Initiator specific NIU, generic NIU, target generic NIU and target specific NIU, as summarized in section Transaction-level concepts. This section goes into more details about the process for read and write incrementing and wrapping bursts and how it can be configured by designers.

请求映射是一系列步骤，涉及 Initiator 专用 NIU、通用 NIU、target 通用 NIU 和 target 专用 NIU。本节详细介绍读写递增/回绕突发的处理过程，以及设计者如何配置这一过程。

> Special cases (fixed and block bursts, exclusive accesses, and so forth.) are documented in the Special bursts section.

特殊情况（固定突发、块突发、独占访问等）在"特殊突发"章节中说明。

> **Note:** There are many documented cases of transaction splitting in this section. Conversely, note that FlexNoC does **not** attempt to combine transactions (it may keep transactions together when necessary, see for example section Exclusive Accesses).

**注意：** 本节记录了大量事务拆分的情况。反之，FlexNoC **不会**尝试合并（combine）事务（必要时会将事务保持在一起，例如独占访问章节所述）。

> For incrementing and wrapping bursts, which are the most common type of bursts, FlexNoC architecture emphasis has been put on interoperability between the various protocols as well as some special processing capabilities, leading to a quite general mapping and splitting process which happens conceptually in steps.

对于最常见的递增突发和回绕突发，FlexNoC 架构重点关注各协议之间的互操作性以及若干特殊处理能力，形成了一套概念上分步进行的通用映射与拆分流程。

---

### Transaction handling steps（事务处理步骤）

> In the following description, steps that happen in the specific NIUs typically apply only to certain protocols, while steps happening in the generic NIU are applicable to all protocols. When multiple steps apply, they apply in sequence.

在以下描述中，发生在专用 NIU 中的步骤通常只适用于特定协议，而发生在通用 NIU 中的步骤适用于所有协议。当多个步骤同时适用时，按顺序依次执行。

> **Note:** Most transactions are not altered during steps 1 to 3, and are simply mapped one to one between the socket protocol and the generic interface, for processing by the initiator generic NIU.

**注意：** 大多数事务在步骤 1～3 中不会被修改，只是在 socket 协议与通用接口之间做一对一映射，由 initiator 通用 NIU 处理。

---

#### Step 1: Narrow burst packing（步骤一：窄突发打包）

> **Applicable to initiator protocols:** AHB, AXI.
> **Configured by:** `protocol: conversion: {minNarrowBurstSize, forceCacheable}`.
> **Step happens in:** initiator specific NIU.

- **适用协议：** AHB、AXI
- **配置参数：** `protocol: conversion: {minNarrowBurstSize, forceCacheable}`
- **发生位置：** initiator 专用 NIU

> When parameter `minNarrowBurstsize` is set to a value other than `None`, the initiator can issue narrow bursts, that is, bursts with an `ASIZE` that is less than the word width. When set to `None`, an assertion occurs and behavior is unpredictable.

当参数 `minNarrowBurstsize` 设置为非 `None` 值时，initiator 可以发出窄突发（narrow burst），即 `ASIZE`（传输位宽）小于字宽（word width）的突发。若设为 `None`，则会触发断言（assertion），行为不可预测。

> With some exceptions, for example, when they are imprecise and non-cacheable, narrow bursts are packed into words, in which case they are split into one precise request per beat. More information can be found in the appropriate specific NIU reference.

除部分例外情况（如非精确且不可缓存的情况），窄突发会被打包为字（word），此时按每节拍（beat）拆分为一个精确请求。详细信息可参见对应专用 NIU 参考文档。

> **Note:** This step implies that narrow bursts will not be transmitted as such at target interfaces, even for target interface protocols that support narrow bursts. As a consequence, the SIZE and LEN will be changed, and the burst may be rounded at the target as a consequence of Step 6: Word alignment.

**注意：** 本步骤意味着窄突发不会以原始形式出现在 target 接口，即使目标接口协议支持窄突发。因此，`SIZE` 和 `LEN` 会被更改，突发可能因步骤六（字对齐）在 target 侧被取整。另见"Non-Cacheable reads（非可缓存读）"章节。

> **Example:** On a 32-bit AHB interface, a cacheable/bufferable burst with ADDR=0x1, LEN=8 beats, SIZE=1 byte is packed into a burst starting at address 0x1 and of length 8 bytes, spanning 3 data words.

**示例：** 在 32 位 AHB 接口上，一个可缓存/可缓冲的突发（ADDR=0x1，LEN=8 节拍，SIZE=1 字节）被打包为从地址 0x1 开始、长度为 8 字节的突发，跨越 3 个数据字。

---

#### Step 2: Splitting into precise bursts（步骤二：拆分为精确突发）

> **Applicable to** initiator protocols AHB and OCP when parameter `burstprecise` is set to 0.
> **Configured by:** for AHB, `useEarlyBurstTermination`, `forceCacheable`, and `rdSplit` and `wrSplit`.
> **Step applies to:** initiator specific NIU.

- **适用条件：** AHB 协议，以及 OCP 协议且参数 `burstprecise` 为 0 时
- **配置参数：** 对于 AHB：`useEarlyBurstTermination`、`forceCacheable`、`rdSplit`、`wrSplit`
- **发生位置：** initiator 专用 NIU

> Imprecise bursts are transactions whose size cannot be predicted by looking at the content of their first request phase after step 1, if applicable. They include:
>
> - AHB non-exclusive INCR bursts
> - Any AHB burst when `useEarlyBurstTermination = True`
> - OCP bursts (`MBurstLength > 1`) when `burstprecise = 0`

非精确突发（imprecise burst）是指在步骤一之后，仅凭第一个请求阶段的内容无法预测其大小的事务。包括：

- AHB 非独占 INCR 突发
- 当 `useEarlyBurstTermination = True` 时的任何 AHB 突发
- 当 `burstprecise = 0` 时的 OCP 突发（`MBurstLength > 1`）

> In the OCP case, imprecise bursts are split into 2-word bursts up to the last transfer (as indicated by `MBurstLength = 1`) which may be alone.

对于 OCP 情况：非精确突发被拆分为每段 2 个字的突发，直到最后一次传输（由 `MBurstLength = 1` 指示），最后一次传输可能单独存在。

> **Example:** A burst with four words respectively with `MBurstLength=3,3,3,1` is split into two fragments, each one comprising two words.

**示例：** 一个 4 字的突发，`MBurstLength` 分别为 3,3,3,1，被拆分为两个片段，每个片段包含 2 个字。

> In the AHB case, cacheable non-narrow non-exclusive INCR imprecise bursts are split into requests containing the number of words specified with parameters `rdSplit` or `wrSplit`. For cacheable reads, this has the effect of prefetching data up to `rdSplit` words, and for cacheable writes, of extending requests up to `wrSplit` words with the associated byte enables set to 0. Non-cacheable transactions may be split into words, or even single beats, depending on NIU parameters.

对于 AHB 情况：可缓存、非窄、非独占的 INCR 非精确突发，被拆分为包含 `rdSplit` 或 `wrSplit` 指定字数的请求。对于可缓存读，效果等同于预取最多 `rdSplit` 个字的数据；对于可缓存写，效果等同于将请求扩展至最多 `wrSplit` 个字，相应字节使能设为 0。不可缓存事务可能被拆分为单字甚至单节拍，取决于 NIU 参数。

> **Example:** With `rdSplit=4`, a cacheable INCR RD burst of 5 words is split into two transactions of 4 words, totaling 8 words.

**示例：** `rdSplit=4` 时，一个 5 字的可缓存 INCR 读突发被拆分为两个各 4 字的事务，共 8 个字。

---

#### Step 3: MRMD to SRMD conversion（步骤三：MRMD 转 SRMD 转换）

> **Applicable to** initiator protocols: AHB, OCP with parameter `burstsinglereq` set to 0.
> **Configured by:** not applicable.
> **Step occurs during:** initiator specific NIU.

- **适用协议：** AHB；OCP 且参数 `burstsinglereq` 为 0 时
- **配置参数：** 不适用
- **发生位置：** initiator 专用 NIU

> For transactions still resulting into bursts (that is, spanning more than one data word) after steps 1 and 2 if applicable, all request phases after the first request phase are dropped.

经过步骤一和步骤二之后，对于仍然形成突发（即跨越多个数据字）的事务，第一个请求阶段之后的所有请求阶段均被丢弃。

> **Example:** On a 32-bit AHB interface without early burst termination, an INCR4 transaction results into a single request phase with the same address as the first AHB beat, and a length of 32 / 8 × 4 = 16 bytes.

**示例：** 在不启用提前突发终止的 32 位 AHB 接口上，一个 INCR4 事务最终只有一个请求阶段，地址与第一个 AHB 节拍相同，长度为 32 / 8 × 4 = 16 字节。

---

#### Step 4: Address decoding and target-based splitting（步骤四：地址解码与基于目标的拆分）

> **Applicable to:** all transactions.
> **Configured by:** Address map, target generic socket `wLen`, `crossBoundary`, `maxWrap` (Architecture: Datapath NIU: Splitting).
> **Step happens in:** initiator generic NIU.

- **适用范围：** 所有事务
- **配置参数：** 地址映射（Address map）、target 通用 socket 的 `wLen`、`crossBoundary`、`maxWrap`（Architecture: Datapath NIU: Splitting）
- **发生位置：** initiator 通用 NIU

> Request address (and optionally address modifiers, such as address spaces or modes) is matched against the system address map, and matching target mapping characteristics are obtained.

请求地址（及可选的地址修饰符，如地址空间或模式）与系统地址映射匹配，并获取匹配目标的映射特性。

本步骤包含以下四个子判断：

##### a. Target MaxWrap（目标最大回绕突发）

> Wrapping bursts whose initial address are not aligned on their size (that is, that are not incrementing bursts) and size exceeds the target `maxWrap` are split into two incrementing bursts, respectively with the wrapping burst address and the initial address. The fragments reenter the full process of step 4.

初始地址未对齐到其大小（即非递增突发）且大小超过目标 `maxWrap` 的回绕突发，被拆分为两个递增突发，分别对应回绕突发地址和初始地址。片段重新进入步骤四的完整流程。

> **Example:** A wrapping burst of 64 bytes starting at address 0x4 sent to a target with `maxWrap=32` is split into an incrementing burst starting at address 0x0 and length 4, followed by an incrementing burst starting at address 0x4 and of length 60.

**示例：** 一个从地址 0x4 开始、长度 64 字节的回绕突发，发往 `maxWrap=32` 的目标时，被拆分为：从地址 0x0 开始长度 4 字节的递增突发，紧跟从地址 0x4 开始长度 60 字节的递增突发。

##### b. Target burst_aligned（目标突发对齐要求）

> Target only accepts incrementing bursts that are a power of two in length, with initial address aligned on their size (applies to OCP targets with `burst_aligned=1`). The burst is split into a first `burst_aligned` fragment, as defined by its initial address and length, and a second fragment. The fragments reenter the full process of step 4.

目标仅接受长度为 2 的幂次、初始地址对齐到其大小的递增突发（适用于 `burst_aligned=1` 的 OCP 目标）。突发被拆分为第一个 `burst_aligned` 片段（由初始地址和长度决定）和第二个片段，均重新进入步骤四流程。

> **Example:** A burst starting at address 0x4 and length 16 bytes is split into a burst starting at address 0x4 and length 4 bytes, followed by a burst starting at address 0x8 and length 12 bytes. Note that in this example the second fragment will be split again by this step when processed.

**示例：** 从地址 0x4 开始、长度 16 字节的突发，被拆分为：从地址 0x4 开始长度 4 字节的突发，以及从地址 0x8 开始长度 12 字节的突发。注意第二个片段在处理时还会被本步骤再次拆分。

##### c. Mapping cross boundary（映射越界检查）

> The mapping `crossBoundary` is a power of 2 which is the smallest of:
>
> - the target protocol `crossBoundary` (4K for AXI, 1K for AHB)
> - the target generic protocol `crossBoundary` setting
> - the size of the matching target address mapping
> - the size of the smallest mapping enclosed in the matching mapping, if any
>
> If the transaction crosses any such boundary, then the transaction is split into a first fragment up to the first crossed boundary, and a remaining fragment which will reenter the full process of step 4.

映射 `crossBoundary` 是 2 的幂次，取以下各项的最小值：

- 目标协议的 `crossBoundary`（AXI 为 4K，AHB 为 1K）
- 目标通用协议的 `crossBoundary` 设置
- 匹配的目标地址映射大小
- 若存在嵌套映射，则取最小嵌套映射的大小

若事务跨越了任一此类边界（即事务的第一个字节与最后一个字节不在同一以 `crossBoundary` 对齐的段内），则事务被拆分为第一个片段（到达第一个越界边界）和剩余片段（重新进入步骤四流程）。

> **Example:** A 32-byte initiator transaction from an AXI interface starting at address 0x3F0 and ending at 0x40F decoded to an AHB target interface is split into two 16-byte transactions, to avoid crossing the AHB 1K boundary constraint.

**示例：** 一个来自 AXI 接口、从地址 0x3F0 到 0x40F 的 32 字节事务，解码到 AHB target 接口时，为避免越过 AHB 1K 边界约束，被拆分为两个 16 字节事务。

##### d. Target mapping maximum burst size（目标映射最大突发大小）

> The target mapping maximum burst size is a power of two, the smallest of:
>
> - `2**(target wLen1)`, that is, the actual maximum power-of-two burst size, in bytes, supported by the target protocol
> - the number of bytes set in the design editor Architecture view Datapath NIU section Splitting page for that particular initiator and target
> - the configured size of the data buffers `nBytePerReadDataBuffer` if the NIU is configured with a reorder buffer

目标映射最大突发大小是 2 的幂次，取以下各项的最小值：

- `2**(target wLen1)`，即目标协议支持的实际最大 2 的幂次突发大小（字节）
- 设计编辑器 Architecture 视图 Datapath NIU 部分 Splitting 页面中针对该 initiator 和 target 配置的字节数
- 若 NIU 配置了乱序缓冲区，则为数据缓冲区 `nBytePerReadDataBuffer` 的配置大小

> The burst start address and size, in bytes, are used to compute the unnecessary number of target words for the transaction, and if this number of words multiplied by the target word width exceeds the target mapping maximum burst size defined above, the transaction is split into:
>
> 1. A first fragment, with the same start address, of maximum size `2**(target wLen1)` bytes.
> 2. Followed by a remaining fragment containing the rest of the transaction which will reenter the full process of step 4.

用突发起始地址和大小（字节）计算事务所需的目标字数，若该字数乘以目标字宽超过上述目标映射最大突发大小，则事务被拆分为：

1. 第一个片段：起始地址不变，最大大小为 `2**(target wLen1)` 字节。
2. 剩余片段：包含剩余事务，重新进入步骤四流程。

> **Example 1:** A 128-byte incrementing burst starting at address 0x10 destined to a 64-bit wide target spans 16 target words. If that target only supports bursts up to 32 bytes, that is, 4 words, then the initiator burst is split into four fragments of 4 words, starting respectively at addresses 0x10, 0x30, 0x50, 0x70.

**示例1：** 从地址 0x10 开始的 128 字节递增突发，目标为 64 位宽，跨越 16 个目标字。若目标仅支持最大 32 字节（即 4 个字）的突发，则 initiator 突发被拆分为 4 个 4 字片段，起始地址分别为 0x10、0x30、0x50、0x70。

> **Example 2:** Re-alignment of incrementing bursts to a target word wider than the initiator can lead to splitting, even if the initiator transaction size does not exceed the burst size capability of the target in bytes: A 128-byte incrementing burst starting at address 0x10 destined to a 256-bit wide target supporting bursts up to 128 bytes (that is, 4 target words) actually spans 5 256-bit target words because of bad alignment, and is split into a fragment of 112 bytes starting at 0x10, followed by a fragment of 16 bytes starting at 0x80.

**示例2：** 递增突发向更宽目标字对齐时，即使 initiator 事务大小未超过目标的突发大小限制（字节），也可能引发拆分：从地址 0x10 开始的 128 字节递增突发，目标为 256 位宽且支持最大 128 字节（4 个目标字）的突发，由于对齐问题实际跨越了 5 个 256 位目标字，因此被拆分为从 0x10 开始的 112 字节片段和从 0x80 开始的 16 字节片段。

> **Note:** The hardware implements the decisions in parallel to speed up the process, but the sequences above indicate the priorities of decisions.

**注意：** 硬件并行执行上述判断以加速处理，但以上顺序反映了判断的优先级。

---

### Default settings for target wLen1 and crossBoundary（wLen1 与 crossBoundary 的默认设置及调优）

> By default, FlexNoC configures the burst cross boundaries, the target maximum incrementing and wrapping burst length according to the target socket parameters, with the exception of ordered targets that do not support bursts at all, for which `wLen1` is set to 7, that is, 128 bytes.

默认情况下，FlexNoC 根据目标 socket 参数配置突发越界边界以及目标最大递增/回绕突发长度。例外情况是：完全不支持突发的有序目标，其 `wLen1` 被设为 7，即 128 字节。

> Such default settings guarantee that no splitting will happen unless a transaction:
>
> - Exceeds its target burst capability, or 128 bytes if that target does not support bursts at all.
> - Attempts to cross the illegal boundaries defined by the target protocol.
> - Crosses a mapping boundary in the address map.

这些默认设置保证：只有当事务满足以下条件之一时才会触发拆分：

- 超过目标突发能力上限，或对于不支持突发的目标，超过 128 字节。
- 试图越过目标协议规定的非法边界。
- 越过地址映射中的映射边界。

> The two first cases enable interoperability between initiators and targets that do not have the same burst length capabilities. That includes IP cores that would use different protocols, but also in some cases IP cores that use the same protocol and different data width: for example a 64-bit AXI3 initiator IP can produce bursts up to 16×8=128 bytes, while a 32-bit AXI3 target IP can only absorb transactions up to 16×4=64 bytes. The splitting mechanism ensures in that case that initiator transactions of more than 64 bytes in length will be split to fragments of a maximum of 64 bytes.

前两种情况保证了突发长度能力不同的 initiator 和 target 之间的互操作性。这不仅包括使用不同协议的 IP，也包括使用相同协议但数据位宽不同的 IP：例如，64 位 AXI3 initiator IP 可以产生最大 16×8=128 字节的突发，而 32 位 AXI3 target IP 最大只能处理 16×4=64 字节的事务。拆分机制确保超过 64 字节的 initiator 事务会被拆分为最大 64 字节的片段。

> The third case ensures a predictable behavior of transactions when the byte address of each byte would not be decoded to the same target: the burst is split into fragments that each are decoded to their target. This is a very desirable mechanism in case of finely interleaved memories.

第三种情况确保了当事务中的各字节地址被解码到不同 target 时的可预测行为：突发被拆分为各自解码到对应目标的片段。这对细粒度交织存储（finely interleaved memory）场景尤为有用。

> There are no reasons to alter this default behavior except in the following cases:
>
> - **Reduce the maximum size of burst** in the systems, in order to increase arbitration opportunities for QoS requirements. This can be achieved by reducing `wLen1` at targets under the Default Setting.
> - **Pre-process bursts for special targets.** Typical examples are: avoid bursts to cross DRAM pages before being submitted to the DRAM controller, split bursts into fragments that fit into a cache line to deal with I/O coherency. This can be achieved by reducing the target `crossBoundary` under the Default Setting.

除以下情况外，无需修改默认行为：

- **减少系统中的最大突发大小**，以增加 QoS 的仲裁机会。可通过在目标端将 `wLen1` 设为低于默认值来实现。
- **针对特殊目标预处理突发。** 典型示例：避免突发在提交给 DRAM 控制器前跨越 DRAM 页，或将突发拆分为符合缓存行大小的片段以处理 I/O 一致性问题。可通过将目标端 `crossBoundary` 设为低于默认值来实现。

> Reducing the `crossBoundary` automatically also limits the maximum burst length, but does not have the same effect: for example setting `crossBoundary=64` at a target generic interface ensures that all transactions will fit into 64-byte cache lines, while setting `wLen1=6` ensures that all transactions do not exceed 64 bytes, but they may still cross addresses that are multiple of 64 bytes.

降低 `crossBoundary` 会自动限制最大突发长度，但两者效果不同：例如，在目标通用接口设置 `crossBoundary=64` 确保所有事务都适配 64 字节缓存行；而设置 `wLen1=6` 确保所有事务不超过 64 字节，但事务仍可能跨越 64 字节对齐地址。

---

#### Step 5: Wrapping burst alignment（步骤五：回绕突发对齐）

> **Applicable to:** all protocols.
> **Configured by:** target `WrapAlign`.
> **Step happens in:** target generic NIU.

- **适用范围：** 所有协议
- **配置参数：** target `WrapAlign`
- **发生位置：** target 通用 NIU

> After step 4, incrementing or wrapping requests are packetized and routed through the fabric to the target network interface. The packet content is not altered by the fabric, except possibly by security firewalls or power management disconnect units, both able to prevent particular transactions to reach their target if a security rule has been violated, or a power domain is not active on the request path.

步骤四之后，递增或回绕请求被分组（packetize）并通过 fabric 路由到目标网络接口。Fabric 不会修改报文内容，除非安全防火墙或断电管理单元阻止了特定事务到达目标（安全规则被违反，或请求路径上的电源域未激活）。

> The target generic NIU performs very few burst transformations:
>
> **Wrapping burst alignment:** The NIU realigns the initial address of a wrapping burst to the target `WrapAlign` parameter, constrained to be the target data word width.
>
> **Note:** This transformation applies only if the target data word width is larger than the initiator data word width.

Target 通用 NIU 执行的突发变换极少：

**回绕突发对齐：** NIU 将回绕突发的初始地址重新对齐到目标 `WrapAlign` 参数，该参数受限于目标数据字宽。

**注意：** 此变换仅在目标数据字宽大于 initiator 数据字宽时生效。

---

#### Step 6: Word alignment（步骤六：字对齐）

> **Applicable to:** word-aligned protocols (nearly all protocols).
> **Configured by:** not applicable.
> **Step happens in:** target specific NIU.

- **适用范围：** 字对齐协议（几乎所有协议）
- **配置参数：** 不适用
- **发生位置：** target 专用 NIU

> If the target protocol constrains burst start address to be aligned to word boundaries (for example, OCP, but not AXI), the transaction address is aligned to a word boundary. If the transfer fits in a single target word, or for write transactions with byte enables this does not necessarily imply rounding since the byte enable pattern can indicate the first valid byte.

若目标协议要求突发起始地址对齐到字边界（例如 OCP，但 AXI 不要求），则事务地址向字边界对齐。若传输适合单个目标字，或对于带字节使能的写事务，这不一定意味着取整，因为字节使能模式可以指示第一个有效字节。

> If the target protocol constrains burst length to be a number of words, the size of the last transfer of the burst is rounded to a full word, and for write transactions with byte enables, the byte enable pattern indicates the last valid byte.

若目标协议要求突发长度为字的整数倍，则突发的最后一次传输大小向上取整为整字，对于带字节使能的写事务，字节使能模式指示最后一个有效字节。

> **Note:** For AMBA protocols, NIUs always use SIZE=word for transactions with multiple beats, so AXI and AHB indeed issue bursts whose length is always a multiple of the word size.

**注意：** 对于 AMBA 协议，NIU 对多节拍事务始终使用 SIZE=word，因此 AXI 和 AHB 发出的突发长度始终是字大小的整数倍。

> **Example:** A 32-bit AXI initiator issues a cacheable RD narrow burst of 4 beats of 1 byte starting at address 0x1, and thus ending at byte 0x4, to a 32-bit AXI target. Target specific NIU issues two beats of 4 bytes, keeping the original address of 0x1, but rounding the end of the burst from original address 0x4 to 0x7. For a WR transaction, the byte enables for bytes 0x5 to 0x7 would be 0. If the target was an OCP instead of AXI, in addition the transaction would start at address 0x0 instead of 0x1, where byte enable of byte 0x0 would be 0 for a WR.

**示例：** 32 位 AXI initiator 向 32 位 AXI target 发起一个可缓存读窄突发：4 节拍，每拍 1 字节，从地址 0x1 开始（到 0x4 结束）。Target 专用 NIU 发出 2 节拍 4 字节的突发，保留原始地址 0x1，但将突发结束地址从 0x4 取整到 0x7。若为写事务，字节 0x5～0x7 的字节使能为 0。若 target 是 OCP 而非 AXI，则事务起始地址变为 0x0（而非 0x1），且写事务中字节 0x0 的字节使能为 0。

---

#### Step 7: Burst splitting or SRMD→MRMD conversion（步骤七：突发拆分或 SRMD→MRMD 转换）

> **Applicable to** target protocols: protocols without bursts (such as APB, OCP with `burstlength` tie-off to 1), MRMD protocols (such as AHB, OCP with `burstsinglereq=0`), or limited burst support (for example, no wrapping bursts).
> **Configured by:** not applicable.
> **Step happens in:** target specific NIU.

- **适用协议：** 无突发协议（如 APB、`burstlength` 固定为 1 的 OCP）；MRMD 协议（如 AHB、`burstsinglereq=0` 的 OCP）；有限突发支持协议（如不支持回绕突发）
- **配置参数：** 不适用
- **发生位置：** target 专用 NIU

> **Burst splitting:** the NIU splits the incoming transactions into single word accesses if the target socket does not support bursts (for example, APB target protocol) or does not support the incoming burst type.
>
> **Note:** Further splitting and processing may happen if the target protocol does not support byte enables, and the incoming burst does not have byte enables fully set. See partial access and byte enables handling section.

**突发拆分：** 若目标 socket 不支持突发（如 APB target 协议），或不支持传入的突发类型，NIU 将事务拆分为单字访问。

**注意：** 若目标协议不支持字节使能，且传入突发的字节使能未全部置位，则可能发生进一步的拆分和处理。详见"部分访问与字节使能处理"章节。

> **Conversion to MRMD:** request address phases are generated for each data word. Write responses status is aggregated and write response is issued to the generic NIU when the last response is obtained from the target.

**转换为 MRMD（Multiple Request Multiple Data）：** 为每个数据字生成请求地址阶段。写响应状态被聚合，当收到目标端最后一个响应时，向通用 NIU 发出写响应。

---

## Endianness handling（字节序处理）

> Endianness of FlexNoC specific NIUs is configured by setting parameters `useBigEndian` or `Endian`, according to the protocol used by the referenced socket (Specification: Interface: NIU: protocol). The parameters set endian operation for the socket.

FlexNoC 专用 NIU 的字节序（endianness）通过参数 `useBigEndian` 或 `Endian` 配置，具体参数取决于所引用的 socket 使用的协议（Specification: Interface: NIU: protocol）。这些参数用于设置 socket 的字节序操作方式。

> **Note:** Endianness is not modified by FlexNoC generic NIUs, or during transport.

**注意：** FlexNoC 通用 NIU 或传输过程中不会修改字节序。

> When big-endian operation is enabled, that is, when the byte address of data bits (7:0) is `(wData / 8) – 1`, data is sent to the corresponding generic NIU without re-encoding.

当大端（big-endian）操作启用时，即数据位 (7:0) 的字节地址为 `(wData / 8) – 1` 时，数据发送到对应通用 NIU 时不做重新编码。

> When big-endian operation is disabled, that is, when the byte address of data bits (7:0) is 0, data is byte-swapped prior to being sent to the FlexNoC generic interface.

当大端操作未启用时，即数据位 (7:0) 的字节地址为 0（小端模式）时，数据在发送到 FlexNoC 通用接口前进行字节交换（byte-swap）。

**相关参数：**

| 参数 | 说明 |
|---|---|
| `useBigEndian` | 启用/禁用大端字节序操作 |
| `Endian` | AHB 相关字节序配置 |

---

## 本章小结

本章详解了 FlexNoC NIU 事务映射与拆分的七个步骤：

| 步骤 | 名称 | 发生位置 | 适用协议 |
|---|---|---|---|
| Step 1 | 窄突发打包（Narrow burst packing） | Initiator 专用 NIU | AHB、AXI |
| Step 2 | 拆分为精确突发（Splitting into precise bursts） | Initiator 专用 NIU | AHB、OCP（`burstprecise=0`） |
| Step 3 | MRMD→SRMD 转换 | Initiator 专用 NIU | AHB、OCP（`burstsinglereq=0`） |
| Step 4 | 地址解码与基于目标的拆分 | Initiator 通用 NIU | 所有协议 |
| Step 5 | 回绕突发对齐（Wrapping burst alignment） | Target 通用 NIU | 所有协议 |
| Step 6 | 字对齐（Word alignment） | Target 专用 NIU | 几乎所有协议 |
| Step 7 | 突发拆分 / SRMD→MRMD 转换 | Target 专用 NIU | APB、AHB、OCP 等 |

**关键参数速查：**

| 参数 | 含义 |
|---|---|
| `minNarrowBurstsize` | 允许的最小窄突发大小 |
| `rdSplit` / `wrSplit` | AHB 读/写拆分字数 |
| `crossBoundary` | 最大允许越界边界（字节，2 的幂次） |
| `wLen1` | 目标最大突发长度（log2，字节） |
| `maxWrap` | 目标支持的最大回绕突发大小 |
| `WrapAlign` | Target NIU 回绕突发对齐粒度 |
| `useBigEndian` | 启用大端字节序 |
