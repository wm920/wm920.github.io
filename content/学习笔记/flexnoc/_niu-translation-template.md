<!--
  FlexNOC NIU 双语对照翻译模板 & 排版规范
  此文件仅供翻译参考，以下划线开头不会被 Hugo 构建
  ============================================================
-->

---
title: "FlexNOC NIU 事务处理（XX）：章节标题 [中英对照]"
date: 2026-05-10
draft: true
series: ["FlexNOC NIU Transaction Handling"]
series_order: X
tags: ["FlexNOC", "NIU", "片上网络", "事务处理"]
categories: ["学习笔记"]
source: "Arteris IP - FlexNOC NIU Transaction Handling Technical Reference (FlexNoC 4.5.1, 2021-02-27)"
description: "一句话摘要，说明本章核心内容。"
---

> **说明**：本系列基于 Arteris IP《FlexNoC NIU Transaction Handling Technical Reference》（FlexNoC 4.5.1，2021年2月27日版）逐章翻译整理，采用中英双语对照排版，原文以引用块呈现，译文紧随其后。信号名、参数名、寄存器字段名保留英文原样。

---

<!--
  ============================================================
  排版规范
  ============================================================

  【1. 段落对照】
  原文段落用 blockquote（> 开头），译文紧跟其后，不加空行隔断：

    > This is the original English paragraph.

    这是对应的中文译文段落。

  多句原文可合并在一个 blockquote 块内，译文整体跟随。

  【2. 标题格式】
  保留原文章节标题，括号内加中文：

    ## Transaction Mapping（事务映射）
    ### Step 1: Address Decoding（步骤一：地址解码）

  【3. 信号 / 参数 / 寄存器名】
  保持英文原样，首次出现时括号内加中文释义：

    AWVALID（写地址有效信号）、ARQOS（读地址 QoS 字段）
    NIU_TARGET_PIPELINE（NIU 目标端流水线参数）

  之后同文中再次出现时无需重复注释。

  【4. 表格处理】
  保留原文表格，列头不翻译；表格下方加译注行说明列义：

    | Signal | Width | Direction | Description |
    |--------|-------|-----------|-------------|
    | AWID   | 8     | Input     | Write transaction ID |

    *译注：Signal=信号名，Width=位宽，Direction=方向，Description=描述*

  【5. 图片插入（矢量图截图）】
  含架构图/流程图的页面渲染为 PNG，存入 images/ 目录：

    ![图1：NIU 在 FlexNOC 中的位置](images/fig-p10.png)
    *图1：NIU（网络接口单元）连接 Master/Slave 与片上网络的架构示意（原文 Figure 1）*

  命名规则：fig-p{PDF页码}.png，如 fig-p10.png、fig-p47.png。

  【6. 注意事项 / 警告框】
  原文 Note/Warning/Caution 块用加粗标识：

    > **Note**：...原文...

    **注意**：...译文...

  【7. 不翻译的内容】
  - 所有信号名、端口名（AWVALID、RDATA、BREADY 等）
  - 参数名（NIU_TARGET_PIPELINE、MAX_OUTSTANDING 等）
  - 协议缩写（AXI、AHB、APB、OCP、CHI、ACE 等）
  - 寄存器字段名
  - 代码块、命令行示例

  ============================================================
-->

## 章节标题（Chapter Title）

> Original paragraph goes here.

译文段落写在这里。

---

### 子章节（Sub-section Title）

> Another original paragraph.

对应译文。

> | Column A | Column B | Column C |
> |----------|----------|----------|
> | Value 1  | Value 2  | Value 3  |

| Column A | Column B | Column C |
|----------|----------|----------|
| Value 1  | Value 2  | Value 3  |

*译注：Column A=字段A，Column B=字段B，Column C=字段C*
