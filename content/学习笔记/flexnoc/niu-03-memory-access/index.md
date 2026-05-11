---
title: "FlexNOC NIU 事务处理（03）：用户信息、内存类型、部分访问与特殊突发 [中英对照]"
date: 2026-05-10
draft: true
series: ["FlexNOC NIU Transaction Handling"]
series_order: 3
tags: ["FlexNOC", "NIU", "片上网络", "内存类型", "字节使能", "posted write", "fixed burst", "2D burst", "OCP", "AXI", "AHB"]
categories: ["学习笔记"]
source: "Arteris IP - FlexNOC NIU Transaction Handling Technical Reference (FlexNoC 4.5.1, 2021-02-27)"
description: "FlexNOC NIU 事务处理系列第三篇，覆盖请求/响应用户信息传输、内存类型与 posted 写、提前写响应、非可缓存读、部分访问与字节使能、固定突发（Fixed burst）及 2D 块突发的完整处理规则。"
---

> **说明**：本系列基于 Arteris IP《FlexNoC NIU Transaction Handling Technical Reference》（FlexNoC 4.5.1，2021年2月27日版）逐章翻译整理，原文以引用块呈现，译文紧随其后。信号名、参数名、寄存器字段名保留英文原样。

---

## Request and response user info transport（请求与响应用户信息传输）

> Most sockets include or optionally include request user information, which are in-band qualifiers attached to the request phases. Typical examples are HPROT for AHB, MReqInfo for OCP, and so on. Such signals can be transported between initiator and targets.

大多数 socket 包含（或可选地包含）请求用户信息，这些信息是附加在请求阶段的带内限定符（in-band qualifier）。典型示例包括 AHB 的 `HPROT`、OCP 的 `MReqInfo` 等。此类信号可以在 initiator 与 target 之间传输。

> **Note:** In some rare cases, request user signals also change the behavior of the initiator specific NIU. This section only discusses how such signals can be transported.

**注意：** 在极少数情况下，请求用户信号也会改变 initiator 专用 NIU 的行为。本节仅讨论此类信号如何被传输。

---

### Request user signals（请求用户信号）

> The general mechanism for handling such Request User info bits is common for all sockets. It involves the definition of `userFlags` and/or `securityFlags` in the design editor Specification view Mappings section User page. Such `userFlags` can transport the value of a request user input from an initiator to a request user output to the slave. In the fabric, `userFlags` are transported as bits in packet headers.

所有 socket 处理请求用户信息位的通用机制相同：在设计编辑器 Specification 视图 Mappings 部分的 User 页面中定义 `userFlags` 和/或 `securityFlags`。`userFlags` 可将 initiator 的请求用户输入值传输到 slave 的请求用户输出。在 fabric 中，`userFlags` 以报文头（packet header）中的比特位传输。

> Note that `userFlags` can be renamed — their name should refer to their functionality, regardless of the particular socket signal names that they are connected to. Socket User signal names can be assigned "alias" in order to facilitate connections and checking.

`userFlags` 可以重命名，名称应反映其功能，而无需与具体 socket 信号名一致。Socket 用户信号名可以设置"别名（alias）"以方便连接与检查。

> **Example:** A `userFlag` named "secure" can more naturally be connected to socket signals whose alias have been set, for example to `HPROT[1]` and `MreqInfo[3]` aliased to "secure".

**示例：** 名为 "secure" 的 `userFlag` 可以更自然地连接到已设置别名的 socket 信号，例如 `HPROT[1]` 和 `MreqInfo[3]` 均别名为 "secure"。

以下约束适用于 `userFlags` 和请求用户信号，必须在 User 页面中填写必要设置：

> - At initiator sockets: a `userFlag` packet bit must be driven by a source from the initiator socket. It can be one of the defined mandatory or optional socket Request User signals, or a tie-off to 0 or 1.
> - At target sockets: a defined mandatory or optional socket Request User signal must be driven from a `userFlag`, a `securityFlag`, or by a tie-off to 0 or 1.

- **Initiator socket 侧：** `userFlag` 报文位必须由 initiator socket 的某个来源驱动，可以是已定义的强制/可选 socket 请求用户信号，或者固定接 0/1。
- **Target socket 侧：** 已定义的强制/可选 socket 请求用户信号必须由 `userFlag`、`securityFlag` 或固定值 0/1 驱动。

> **Note:** There is no difference in the handling of `userFlags` and `securityFlags`, except that `securityFlags` can also be used to qualify security of transactions during address decode stage.

**注意：** `userFlags` 和 `securityFlags` 的处理方式相同，唯一区别是 `securityFlags` 还可以在地址解码阶段用于限定事务的安全性。

> If a transaction is split, all request packets and in consequence transaction fragments at targets share the same request User signal values.

若事务被拆分，所有请求报文（以及 target 侧的所有事务片段）共享相同的请求用户信号值。

> **Note:** The polarity of a user request signal cannot be inverted in a NIU before or after being transported in the fabric. The only exception is AHB `HProt[0]`, which is always inverted in both initiator and target NIUs, in order to provide a `userFlag` that has the same polarity as AXI `AxPROT[0]`.

**注意：** 用户请求信号的极性在 NIU 中或 fabric 传输前后均不能取反。唯一例外是 AHB 的 `HProt[0]`，在 initiator 和 target NIU 中始终取反，以使对应 `userFlag` 与 AXI 的 `AxPROT[0]` 极性一致。

---

### Response user signals（响应用户信号）

> Some sockets optionally include response user information, which are in-band qualifiers attached to the response phases. Typical examples are `SRespInfo` for OCP, and proprietary extensions for AMBA.

部分 socket 可选地包含响应用户信息，这些信息是附加在响应阶段的带内限定符。典型示例为 OCP 的 `SRespInfo` 以及 AMBA 的专有扩展。

> Such response user bits are typically a copy of a user bit associated with the corresponding request, and can be used by the initiator IP to have information about particular aspects of the response, without having to remember such information since the request — the NIUs provide that storage functionality.

此类响应用户位通常是对应请求中某个用户位的副本，initiator IP 无需从请求时起就记忆这些信息即可获得响应的特定方面信息，NIU 提供了相应的存储功能。

> **Example:** A request user bit can be used to indicate that the current request is the last transaction in a series of related transactions, and copied into a response user bit — receiving this bit set in the response indicates that the last transaction of the series has been processed. This can be particularly useful in the presence of out-of-order responses.

**示例：** 请求用户位可用于标记当前请求是一系列相关事务中的最后一个，并复制到响应用户位——在响应中收到该位置位，表示该系列的最后一个事务已被处理完成。这在存在乱序响应的场景中尤为有用。

响应用户信息的通用处理机制：

> - At initiator sockets: a Response User signal must be driven by a source from the initiator socket. It can be one of the defined mandatory or optional socket Request User signals, or a tie-off to 0 or 1.
> - At target sockets: no support.

- **Initiator socket 侧：** 响应用户信号必须由 initiator socket 的某个来源驱动，可以是已定义的强制/可选 socket 请求用户信号，或固定值 0/1。
- **Target socket 侧：** 不支持。

> Since the loopback mechanism does not involve `userFlags`, it is configured directly at the initiator socket level, using the conversion `rspInfoMap` parameters.

由于回环（loopback）机制不涉及 `userFlags`，因此直接在 initiator socket 层面通过转换参数 `rspInfoMap` 配置。

> **Note:** The same request user signal may be used both as a source for `userFlag` and a source for a response user signal.

**注意：** 同一个请求用户信号可以同时作为 `userFlag` 的源和响应用户信号的源。

> **Note:** For read bursts, the response user flag is duplicated on all responses. In general the response user mechanism is not supported for MRMD initiators.

**注意：** 对于读突发，响应用户标志会在所有响应上复制。通常，MRMD initiator 不支持响应用户机制。

---

## Memory types, posted writes and early write responses（内存类型、posted 写与提前写响应）

> This section discusses how NIUs are sensitive to memory type information passed with requests, and how this affects read and write transaction processing.

本节讨论 NIU 如何感知随请求传递的内存类型信息，以及这如何影响读写事务的处理。

---

### NoC write responses（NoC 写响应）

> Inside the NoC, all write transactions have a response (that is, there is always a response from a target generic NIU to the initiator generic NIU, and even to the initiator specific NIU). Mandatory write responses in the fabric are necessary for example to track the activity state of power domains. In addition, receiving a write response at the initiator NIU provides a guarantee that all the write data has been accepted by the target NIU of the transaction.

在 NoC 内部，所有写事务均有响应（即 target 通用 NIU 到 initiator 通用 NIU，乃至 initiator 专用 NIU 始终有响应）。Fabric 中强制的写响应是必要的，例如用于追踪电源域的活动状态。此外，在 initiator NIU 收到写响应，保证所有写数据已被该事务的 target NIU 接收。

对于 initiator 为 SRMD 协议（即每个事务只有一个写响应）的情况：

> - Except when early write responses have been activated at initiator NIU, the initiator IP receives a write response only after both:
>   - All data has been committed at the target NIU.
>   - The write response has been received from the target, assuming the target socket protocol provides a write response.
> - Only initiator specific NIUs may produce early write responses, or delete unnecessary responses for the initiator protocol.
> - Target NIUs connected to target socket protocols that do not produce write responses issue their own responses after the last data has been accepted by the target NIU.
> - For initiator MRMD protocols, the above rules are only valid for the last write response of the transaction: the write responses for the first data may be issued before the data has actually reached the target, and should be considered as "early response" with no meaning.

- 除非 initiator NIU 激活了提前写响应，否则 initiator IP 只有在以下两个条件都满足后才会收到写响应：
  - 所有数据已提交到 target NIU。
  - 已从 target 收到写响应（前提是 target socket 协议提供写响应）。
- 只有 initiator 专用 NIU 可以生成提前写响应，或删除 initiator 协议不需要的响应。
- 连接到不产生写响应的 target socket 协议的 target NIU，在 target NIU 接受最后一个数据后自行发出响应。
- 对于 initiator 为 MRMD 协议的情况，上述规则仅对事务的最后一个写响应有效：第一个数据的写响应可能在数据实际到达 target 之前就发出，应视为"无意义的提前响应"。

---

### Early write response support（提前写响应支持）

> When FlexNoC initiator NIUs use protocols that support posted-write semantics, the corresponding specific NIUs can issue early write responses in certain cases. Posted-write semantics include OCP WR and WNP transactions, and AMBA-protocol bufferable bits.

当 FlexNoC initiator NIU 使用支持 posted-write 语义的协议时，对应的专用 NIU 在某些情况下可以发出提前写响应（early write response）。Posted-write 语义包括 OCP 的 `WR` 和 `WNP` 事务，以及 AMBA 协议的 bufferable 位。

---

### Non-cacheable reads（非可缓存读）

> Some socket protocols, mostly AMBA, can provide a cacheable attribute with transactions. Non-cacheable read transactions should not be rounded, since reading extra bytes may destroy them.

部分 socket 协议（主要是 AMBA）可以为事务提供可缓存（cacheable）属性。非可缓存读事务不应被取整（round），因为读取额外字节可能破坏数据。

NoC 中可能发生事务取整的步骤有两处：

> - At initiator NIUs, when prefetching imprecise bursts (AHB).
> - When rounding to target words at target specific NIUs.

- 在 initiator NIU 处，预取（prefetch）非精确突发（AHB）时。
- 在 target 专用 NIU 处，向目标字取整时。

> The first case is taken care of by disabling prefetching when the transaction is marked as non-cacheable.

第一种情况：当事务被标记为非可缓存时，通过禁用预取来处理。

> However, rounding bursts to target words is unavoidable for some protocols, and furthermore the amount of rounding depends on possible data width conversion between the initiator and target.

然而，向目标字取整对某些协议而言是不可避免的，且取整量取决于 initiator 与 target 之间可能存在的数据位宽转换。

> Non-cacheable access from an initiator protocol to the same target protocol of the same width will not produce rounding. When the target protocol or data width differs, some rounding may happen even for non-cacheable reads.

相同协议、相同位宽时，非可缓存访问不会产生取整。当目标协议或数据位宽不同时，即使是非可缓存读也可能发生取整。

---

## Partial accesses and byte enables（部分访问与字节使能）

> Partial accesses are transfers that encompass less than a full socket data word. They typically require mechanisms to indicate the bytes that are active on that particular transfer, such as byte enables, or SIZE field for AMBA.

部分访问（partial access）是指传输量小于一个完整 socket 数据字（word）的传输。它们通常需要某种机制来标识该次传输中活跃的字节，例如字节使能（byte enable）或 AMBA 的 `SIZE` 字段。

> In addition, byte enables can be used to indicate that some particular bytes should not be written, even during long bursts.

此外，字节使能也可用于指示某些特定字节不应被写入，即使在长突发中也适用。

> Protocols have different ways to indicate partial access and/or write byte enable, and sometimes no way, leading to particular issues when an initiator attempts to send such an access to a target that does not support it.

各协议指示部分访问和/或写字节使能的方式各异，有时甚至没有该机制，这在 initiator 向不支持该特性的 target 发送此类访问时会引发特殊问题。

> **Note:** The notion of partial access is local to an interface, that is, what appears as a partial access on a 128-bit initiator interface may appear as full word on a 64-bit interface, or even a burst on a 32-bit target interface. In the following description, partial access must always be understood in the context of the initiator interface or target interface, independently.

**注意：** 部分访问的概念是相对于接口的：在 128 位 initiator 接口上看起来是部分访问，在 64 位接口上可能是完整字访问，在 32 位 target 接口上甚至可能是突发。以下描述中，部分访问始终应在 initiator 接口或 target 接口各自的上下文中理解。

---

### Initiator partial accesses（Initiator 侧部分访问）

> Initiator NIUs accurately process partial accesses for protocols that provide them as an initial byte address, and an access size — for example AMBA AHB and AXI.

对于以初始字节地址和访问大小（access size）表示部分访问的协议（如 AMBA AHB 和 AXI），initiator NIU 能精确处理部分访问。

> For protocols such as OCP that provide a word address and a byte enable pattern, byte enable patterns that are force-aligned (that is, corresponding to a half-word, quarter-word, and so forth, down to a byte, properly aligned to their size in bytes), are accurately processed. Byte enable patterns that do not fit these constraints are rounded to the smallest enclosing force-aligned pattern by the initiator specific NIU before being issued to the generic NIU.

对于以字地址和字节使能模式（byte enable pattern）表示部分访问的协议（如 OCP），强制对齐（force-aligned）的字节使能模式（即对应半字、四分之一字等，按其字节大小正确对齐的模式）可以被精确处理。不符合此约束的字节使能模式，在发往通用 NIU 之前，由 initiator 专用 NIU 取整为最小的包含该模式的 force-aligned 模式。

> **Example:** On AXI 64-bit initiator interface, a partial transfer with `ALEN=0`, `ASIZE=4` bytes, `ADDR=0x1` is processed as starting at byte address 0x1 and length 3 bytes. An equivalent OCP partial access with `MAddr=0x0` and `MByteEn=0xE` is rounded to the enclosing force-aligned pattern `0xF`, and is processed starting at byte address 0x0 and length 4 bytes.

**示例：** 在 64 位 AXI initiator 接口上，`ALEN=0`、`ASIZE=4` 字节、`ADDR=0x1` 的部分传输被处理为从字节地址 0x1 开始、长度 3 字节。等价的 OCP 部分访问（`MAddr=0x0`，`MByteEn=0xE`）被取整为包含它的 force-aligned 模式 `0xF`，处理为从字节地址 0x0 开始、长度 4 字节。

> A special case of read partial access is null read, that is, read with no byte enable set (legal for example in OCP). The initiator generic NIU can only process such empty transactions without rounding if the initiator generic interface parameter `usePreNullRead` has been set to `True`, otherwise it rounds the transaction to a single read byte.

读部分访问的特殊情况是**空读（null read）**，即不设置任何字节使能的读（例如在 OCP 中合法）。只有当 initiator 通用接口参数 `usePreNullRead` 设为 `True` 时，initiator 通用 NIU 才能不取整地处理此类空事务；否则将事务取整为单字节读。

> Target generic NIUs receiving such read null transactions drop them, unless their generic interface has been set with `usePreNullRead=True` as well (only possible for target protocols supporting null read transactions such as OCP with byte enables), in which case the null transaction appears on the target socket.

收到此类空读事务的 target 通用 NIU 会将其丢弃，除非其通用接口也设置了 `usePreNullRead=True`（仅适用于支持空读事务的 target 协议，如支持字节使能的 OCP），此时空事务会出现在 target socket 上。

> **Note:** Some OCP cores also use Null reads as barrier transaction requests. See OCP NIU reference for details.

**注意：** 某些 OCP core 还将空读用作屏障（barrier）事务请求，详见 OCP NIU 参考文档。

---

### Initiator write byte enable（Initiator 写字节使能）

> Initiator NIUs accurately support initiator write byte enables if available from the protocol; each write data byte enable is sent to the target NIU together with the associated data byte.

若协议提供写字节使能，initiator NIU 能精确支持，每个写数据字节使能与对应的数据字节一同发送到 target NIU。

> If an initiator protocol does not provide data byte enables, they are considered to be 1.

若 initiator 协议不提供数据字节使能，则默认视为全部为 1（全使能）。

---

### Target partial accesses（Target 侧部分访问）

> After processing by an initiator NIU, an initiator transaction, or a fragment of an initiator transaction, may appear as a partial access at the target.

经 initiator NIU 处理后，initiator 事务或其片段在 target 侧可能表现为部分访问。

> **Note:** It is not possible to set `crossBoundary` or `wLen1` at the target to a smaller value than the data word width, so an initiator transaction will never be split systematically into fragments smaller than the target word. However, depending on transaction size and alignment, the first fragment and/or the last fragment of an initiator transaction may result in target partial accesses.

**注意：** target 端的 `crossBoundary` 或 `wLen1` 不能设为小于数据字宽，因此 initiator 事务不会被系统性地拆分为小于 target 字宽的片段。但根据事务大小和对齐情况，initiator 事务的第一个和/或最后一个片段可能在 target 侧形成部分访问。

> With a few exceptions (AXI being one because it allows unaligned start transfers), read partial accesses at target are rounded in target specific NIUs to the smallest power-of-two aligned sub-word of the target.

除少数例外（AXI 允许非对齐起始传输），target 侧的读部分访问在 target 专用 NIU 中被取整为最小的 2 的幂次对齐子字。

> Likewise, write partial transfer requests are rounded, but in most cases this does not imply writing extra bytes, assuming that the target supports write byte enables.

类似地，写部分传输请求也会被取整，但在大多数情况下（假设 target 支持写字节使能）这并不意味着会写入额外字节。

> **Example:** On AXI 64-bit target interface, a partial transfer at byte address 0x1 with length 3 bytes is converted to a partial access with `ALEN=0`, `ASIZE=4` bytes, and `ADDR=0x1`, that is, identical to the initiator access. On an OCP 64-bit target with byte enables, it is converted into a partial access with `MAddr=0x0` and `MByteEn=0xF`, that is, rounded to four bytes.

**示例：** 在 64 位 AXI target 接口上，字节地址 0x1、长度 3 字节的部分传输，被转换为 `ALEN=0`、`ASIZE=4` 字节、`ADDR=0x1` 的部分访问，与 initiator 访问完全相同。在带字节使能的 64 位 OCP target 上，则转换为 `MAddr=0x0`、`MByteEn=0xF`，即取整为 4 字节。

---

### Target write byte enable（Target 写字节使能）

> The behavior of target NIUs receiving requests with corresponding data bytes not having all their byte enables at 1, and whose target protocol does not support write byte enables, or only supports restricted sets of byte enable patterns, differs based on the target socket protocol and transaction length.

当 target NIU 收到的请求中有数据字节的字节使能不全为 1，且 target 协议不支持写字节使能（或仅支持受限的字节使能模式集合）时，其行为因 target socket 协议和事务长度而异。

详细的 target NIU 行为以及可能产生的带内错误，请参见各 target NIU 参考文档。

---

## Special bursts（特殊突发）

> For fixed and 2D-type bursts, which are much less widely used than incrementing and wrapping bursts, Arteris FlexNoC architecture privileges simplicity over interoperability, with focus on typical use cases. Consequently, the number of possible transformations, such as splitting or width conversion, are very limited for these burst types.

对于固定突发（fixed burst）和 2D 类型突发，相较于递增突发和回绕突发，Arteris FlexNoC 架构在互操作性和简洁性之间选择了后者，聚焦典型使用场景。因此，这些突发类型可能发生的变换（如拆分或位宽转换）非常有限。

> Special bursts that are marked as exclusive accesses by the initiator are considered illegal, and treated as initiator protocol violations (simulation assertion, undefined silicon behavior).

被 initiator 标记为独占访问（exclusive access）的特殊突发被视为非法，按 initiator 协议违规处理（仿真断言，硅片行为未定义）。

> **Note:** The Arteris FlexNoC standard term **fixed burst** is equivalent to **streaming burst** in OCP terminology.

**注意：** Arteris FlexNoC 标准术语中的**固定突发（fixed burst）**等价于 OCP 术语中的**流式突发（streaming burst）**。

---

### Fixed bursts（固定突发）

> Fixed bursts can be issued for the OCP protocol by setting parameter `burstseq_strm_enable` to 1 or by AXI initiators.

固定突发可由 OCP 协议（设置参数 `burstseq_strm_enable` 为 1）或 AXI initiator 发出。

> Fixed burst support for AXI can be disabled by setting protocol group parameter `useFixed`. When an initiator NIU configured with `useFixed` set to `False` receives a fixed transaction, it issues an assertion, and behavior is undefined. The same configuration for a target NIU returns an in-band error if the NIU receives a fixed burst from an initiator.

AXI 的固定突发支持可通过协议组参数 `useFixed` 禁用。当 `useFixed` 设为 `False` 的 initiator NIU 收到固定事务时，触发断言，行为未定义。对 target NIU 做同样配置时，若收到来自 initiator 的固定突发，则返回带内错误。

#### Fixed burst restrictions（固定突发限制）

> For fixed bursts, the transaction fixed width is defined as the width of each initiator transfer part of the burst (`AxSIZE` of AXI, defined by byte enable pattern for OCP). The transaction length is defined by the number of transfers (beats for AXI, burst length for OCP). As a rule, neither the transaction width nor the length may change between the initiator and target socket for fixed bursts.

对于固定突发：**事务固定宽度**定义为突发中每次 initiator 传输的宽度（AXI 的 `AxSIZE`，OCP 由字节使能模式定义）；**事务长度**定义为传输次数（AXI 的节拍数，OCP 的突发长度）。原则上，固定突发的事务宽度和长度在 initiator 与 target socket 之间均不得改变。

FlexNoC 互连中适用于固定突发的限制（按顺序应用，若某条件引发错误则不再评估后续条件）：

> 1. Initiator fixed bursts of transaction fixed width greater than the target word width are unsupported and result in an in-band error being returned to the initiator.

1. 事务固定宽度大于 target 字宽的 initiator 固定突发不受支持，返回带内错误给 initiator。

> 2. If the initiator NIU is configured with a reorder buffer, fixed bursts of transaction fixed length greater than `8 × nBytePerReadBuffer / wData(target)` are unsupported, and result in an in-band error being returned to the initiator.

2. 若 initiator NIU 配置了乱序缓冲区（reorder buffer），事务固定长度大于 `8 × nBytePerReadBuffer / wData(target)` 的固定突发不受支持，返回带内错误。

> 3. Initiator fixed bursts are not supported to target with address interleaving granularity below 128 bytes.

3. 不支持目标地址交织粒度低于 128 字节时的 initiator 固定突发。

> 4. Initiator fixed bursts of transaction length greater than the maximum target NIU burst capability are unsupported, and result in an in-band error being returned to the initiator.
>    More precisely, an error is returned if the transaction length exceeds `target 2^wLen1 / (wData / 8)`, which is equivalent to the target being able to accept bursts of as many words as the fixed transaction length without splitting them.

4. 事务长度超过 target NIU 最大突发能力的 initiator 固定突发不受支持，返回带内错误。
   更精确地说，若事务长度超过 `target 2^wLen1 / (wData / 8)`（即 target 能不经拆分地接受与固定事务长度等量的字数），则返回错误。

> **Example:** A 128-bit AXI initiator can issue 16-beat fixed bursts to a 32-bit AXI target, also capable of 16-beat fixed bursts. The maximum fixed transaction length on the initiator side is 16, and a setting for target parameter `wLen1` of 6 is sufficient for the 2^6/(4) = 16-beat fixed bursts to be accepted, of a fixed width that is necessarily four bytes or less. For the same initiator to issue 16-beat fixed bursts to a 32-bit APB target, the APB target generic interface must likewise be configured with parameter `wLen1` set to 6 or greater.

**示例：** 128 位 AXI initiator 向 32 位 AXI target（同样支持 16 节拍固定突发）发出 16 节拍固定突发。initiator 侧最大固定事务长度为 16，target 参数 `wLen1=6` 即可满足条件（2^6/(4)=16 节拍），固定宽度必须为 4 字节或更小。若相同 initiator 向 32 位 APB target 发出 16 节拍固定突发，APB target 通用接口也必须将 `wLen1` 设为 6 或更大。

> 5. For OCP initiators, only fixed bursts with a fixed width equal to a power of two, and associated byte enable pattern force-aligned, are accurately supported. Issuing fixed bursts with a non force-aligned byte enable pattern will result in undesirable padding of each word to an enclosing force-aligned byte enable pattern.

5. 对于 OCP initiator，只有固定宽度为 2 的幂次且字节使能模式 force-aligned 的固定突发才能被精确支持。发出字节使能模式非 force-aligned 的固定突发将导致每个字被填充（padding）为包含该模式的 force-aligned 字节使能模式，产生不期望的副作用。

> 6. Initiator fixed bursts to a target that does not support them results in an in-band error being returned to the initiator, unless protocol conversion group parameter `convertFixedToSingle` is set to `True` for that target.

6. 发往不支持固定突发的 target 的 initiator 固定突发将返回带内错误，除非针对该 target 的协议转换组参数 `convertFixedToSingle` 设为 `True`。

> **Note:** The `convertFixedToSingle` parameter appears for a given target only if at least one initiator may issue Fixed transactions to that particular target. This parameter should be set to `True` to enable the splitting of Fixed burst and conversion into single word accesses with identical addresses for that target.

**注意：** `convertFixedToSingle` 参数仅在至少有一个 initiator 可能向该 target 发出固定事务时出现。将其设为 `True` 可将固定突发拆分并转换为目标端的单字访问（地址相同）。这使得向不支持固定突发语义的 target 协议（如 APB）发送固定突发成为可能，但需注意固定宽度和长度的前述限制仍然有效，且原始突发的固定性质在转换过程中已丢失。

> **Note:** When parameter `convertFixedToSingle` is set to `True`, the target NIU waits for all pending transactions to drain before processing an incoming fixed burst.

**注意：** 当 `convertFixedToSingle` 设为 `True` 时，target NIU 在处理传入的固定突发前，会等待所有挂起事务排空。

> When the fixed width is less than the target word width, then the target protocol must offer a mechanism to reduce the width of each word to the fixed width (such as byte enables for OCP, or SIZE for AMBA), even if parameter `convertFixedToSingle` is set, otherwise, an in-band error is returned.

当固定宽度小于 target 字宽时，target 协议必须提供将每个字宽度减小到固定宽度的机制（如 OCP 的字节使能，或 AMBA 的 `SIZE`），即使设置了 `convertFixedToSingle`，否则返回带内错误。

> **Note:** OCP support of streaming bursts is restricted to OCP protocol configurations including byte enables, in order to support fixed bursts with fixed widths inferior to one word. In addition, target NIUs support conversion of fixed bursts to MRMD OCP configurations even if parameter `burstseq_strm_enable` is set to 0.

**注意：** OCP 对流式突发的支持仅限于包含字节使能的 OCP 协议配置，以支持固定宽度小于一个字的固定突发。此外，即使参数 `burstseq_strm_enable` 设为 0，target NIU 也支持将固定突发转换为 MRMD OCP 配置。

> **Note:** Fixed bursts are the only AXI bursts, that is, transactions with signal `ALEN` greater than 1 word, at target NIUs where `ASIZE` may not indicate a full data word; it will be set to the fixed width.

**注意：** 固定突发是唯一一种在 target NIU 处 `ASIZE` 可能不表示完整数据字的 AXI 突发（即 `ALEN` 大于 1 字的事务）；此时 `ASIZE` 将被设为固定宽度。

**相关参数：**

| 参数 | 说明 |
|---|---|
| `useFixed` | 启用/禁用 AXI 固定突发支持 |
| `nBytePerReadBuffer` | 乱序缓冲区每个读数据缓冲区的字节数 |
| `convertFixedToSingle` | 将固定突发转换为单字访问 |

---

### Block (2D) bursts（块（二维）突发）

> 2D bursts are only supported on OCP protocol (with `burstseq_blck_enable=1`). They may however be issued to OCP MRMD targets not supporting 2D bursts, in which case they will be split into single accesses. This can be typically used to issue 2D bursts to SRAMs.

2D 突发仅在 OCP 协议上受支持（需设置 `burstseq_blck_enable=1`）。但可以向不支持 2D 突发的 OCP MRMD target 发出，此时将被拆分为单字访问，典型用途是向 SRAM 发出 2D 突发。

2D 突发事务从 initiator IP 发送到 target IP 的规则（按顺序应用）：

> 1. All initiators and targets that send or receive 2D bursts in a design must be OCP protocols with the same socket data width, greater than or equal to 64 bits.

1. 设计中所有发送或接收 2D 突发的 initiator 和 target 必须都是 OCP 协议，且 socket 数据位宽相同，不低于 64 位。

> 2. The maximum size, in bytes, of a 2D burst that is supported by an OCP initiator NIU is the same as for incrementing bursts, that is: `MburstLength × MBlockHeight <= 2^(burstlength_wdth) – 1`. If an initiator attempts to send a larger 2D burst, behavior is undefined.

2. OCP initiator NIU 支持的 2D 突发最大字节数与递增突发相同：`MburstLength × MBlockHeight <= 2^(burstlength_wdth) – 1`。若 initiator 尝试发送更大的 2D 突发，行为未定义。

> 3. When parameter `usePreBlck` is set to `False` at target NIU generic interface, or when the size of the 2D burst in bytes exceeds the target `2^wLen1`, an in-band error is returned.

3. 当 target NIU 通用接口的 `usePreBlck` 设为 `False`，或 2D 突发字节数超过 target 的 `2^wLen1` 时，返回带内错误。

> 4. If the target OCP socket supports 2D bursts, but the transaction exceeds the target Height, Length, or Stride capacity, the transaction is split into BLCK accesses of Length and Height 1 by the specific NIU.

4. 若 target OCP socket 支持 2D 突发，但事务超过 target 的 Height、Length 或 Stride 容量，则由专用 NIU 将事务拆分为 Length 和 Height 均为 1 的 BLCK 访问。

> 5. If the target OCP does not support 2D bursts but is MRMD, the transaction is split by the target specific NIU into single-word accesses.

5. 若 target OCP 不支持 2D 突发但为 MRMD 模式，则由 target 专用 NIU 将事务拆分为单字访问。

---

## 本章小结

| 特性 | 关键说明 |
|---|---|
| `userFlags` / `securityFlags` | 在 initiator→target 路径上传输请求用户信息位，fabric 以报文头比特承载 |
| 响应用户信号 | 仅 initiator 侧支持，通过 `rspInfoMap` 参数配置回环，target 侧不支持 |
| 写响应 | NoC 内部所有写事务均有响应；initiator 收到写响应表示数据已提交到 target NIU |
| Posted write / Early response | AMBA bufferable 或 OCP WR/WNP 时，专用 NIU 可提前返回写响应 |
| 非可缓存读 | 禁止在 initiator 侧预取；target 侧取整对相同协议相同位宽不生效，跨位宽可能发生 |
| 部分访问 | AMBA 以地址+SIZE 表示；OCP 以字节使能模式表示，非 force-aligned 时自动取整 |
| 空读（null read） | 需双端 `usePreNullRead=True`；否则取整为单字节读或被丢弃 |
| 固定突发（Fixed burst） | 宽度和长度不得跨接口改变；违规时返回带内错误；`convertFixedToSingle` 可转换为单字访问 |
| 2D（块）突发 | 仅 OCP 支持（`burstseq_blck_enable=1`）；所有相关接口须相同位宽且 ≥64 位 |
