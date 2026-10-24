---
title: DeepSeek V4.1 Flash 推理部署问答（中文记录）
description: DeepSeek 官方《DeepSeek V4.1 Flash Inference Deployment Q&A》的中文记录：开源背景、系统架构、集群与存储、并行与缓存、容错、低精度、内核、投机解码与 Recipe。
weight: 3
---

本文是 DeepInfer 仓库内 `doc/inference_deployment_qa.md` 的中文记录，按原文结构逐节翻译。环境变量、参数名、数字、专有名词保留原文写法；如译文与英文原文有冲突，以[英文原文](https://github.com/deepseek-ai/DeepInfer/blob/main/doc/inference_deployment_qa.md)为准。

> 记录时点：2026-10-24。原文是 DeepSeek 针对 V4.1 Flash 推理系统发布的部署问答，随仓库持续更新。原文采用 Apache 2.0 许可；本文作为中文记录同样遵循该许可，并保留原文链接与仓库出处。

## 为什么需要这份问答？（编者说明）

README 回答「支持什么、怎么跑」，这份问答回答「为什么这样设计、数字在什么条件下成立」。两者缺一不可：

- 只看 README，容易把 155 万亿 token/天、每 GPU 小时 38 美元当成通用性能结论；问答会告诉你这些是生产集群在特定硬件、特定流量结构下的指标。
- 只看性能结论，容易忽略架构前提：E/P/D 分离、Wide-EP、3FS 上的 KV Cache、EPLB 与两级负载均衡，缺一块都可能得不到同样的结果。
- 问答还明确列出「哪些能力没开源」：KV Cache 管理与负载均衡和内部基础设施耦合，完整在线服务仍需自建。

| 项 | 内容 |
|----|------|
| 原文 | `deepseek-ai/DeepInfer` 仓库 `doc/inference_deployment_qa.md` |
| 记录范围 | 开源背景、系统总览、集群与存储、EPLB、DP 负载均衡、KV Cache、扩缩容与容错、低精度、内核、投机解码、异步调度、结构化输出、Recipe |
| 记录时点 | 2026-10-24 |
| 许可 | Apache License 2.0 |
| 翻译原则 | 保留问答结构；数字、环境变量、格式名不改写；冲突时以英文原文为准 |

> ### 💡 新人小结（编者注）：这份问答的正确读法
>
> - 它描述的是 **DeepSeek 自己的生产系统**，不是一台通用服务器的调优手册。
> - 看到数字先找三个限定词：**什么硬件、什么模型、什么流量**。
> - 看到「提升了多少」先问：**和什么基线比、在哪个并发区间**。
> - 看到「未开源」要记下来：这些是你自建完整服务时必须补的工程。

---

## 一、开源背景与总体目标

> 模型定义能力的边界；而把这些能力以合理成本服务给海量用户，是**推理框架**必须解决的核心问题。

在大规模 AI 需求下，推理框架已经从单机服务程序演变为跨集群协同的复杂系统。内核、通信、调度与缓存管理相互作用，共同决定端到端性能。对大规模在线服务而言，性能优化必须建立在具体部署条件上：用什么芯片、硬件如何组织、计算如何分布、此前计算的结果如何复用。

DeepSeek V4.1 Flash 推理系统从开始就围绕三个目标构建：

1. **支持多家加速器**：在不同硬件平台上运行，并充分利用各平台性能。
2. **围绕通用硬件组织 Prefill 与 Decode**：Prefill 使用普通的 8 卡机器，Decode 使用普通的 64–128 卡超节点；计算、通信与调度围绕两个阶段的负载特征设计。
3. **KV Cache 保留超过 72 小时**：为上下文复用提供更长窗口，命中时减少重复 Prefill 计算，同时考虑长保留带来的容量与管理开销。

围绕这些目标，团队持续优化计算、通信、调度与存储，在满足延迟要求的前提下提高系统有效吞吐——用更少算力与能源服务更多请求，让智能服务以更低价格触达更多用户。

DeepSeek 此前已开源 [TileLang 与一系列推理内核库](https://mp.weixin.qq.com/s/X41mKH4Ds-VXUAnK6M8Eww)。本次开源 **DeepInfer**，即面向大规模云服务的推理框架。它支撑 DeepSeek V4.1 Flash 的高吞吐、低延迟大规模推理服务。官方披露：**单个集群当前每天服务 155 万亿 token，按谷时电价折合每 GPU 小时收入 38 美元**。

DeepInfer 将高性能内核与深度优化的推理引擎结合，支持分布式推理、投机解码、SWA bounded replay、结构化输出等能力，并同时运行在 NVIDIA 与华为昇腾平台。同步开源的 **deepseek-recipe** 是生产级 Token 编解码组件，在主流 LLM API 协议与模型 token 序列之间高效、准确地转换；其处理逻辑与 DeepSeek 官方 API 完全对齐，第三方部署可以获得与官方一致的 API 行为。未来 DeepSeek 模型的推理代码及后续性能工作，将继续在 DeepInfer 中开源。

推理系统的 KV Cache 管理与负载均衡组件与内部基础设施耦合很深，这部分代码尚未完全开源。官方发布的[知乎长文](https://zhuanlan.zhihu.com/p/2095560295930508953)从上述设计目标出发，描述了 V4.1 Flash 推理系统的整体架构与关键实现，包括这两个组件的设计。本问答以该长文为基础，并将持续更新。

## 二、推理系统总览

**优化目标是更高吞吐与更低延迟。** 为此采用：

- **MoE 部分在（超）节点内使用专家并行（EP）**：提高 batch size 与 GPU 矩阵乘效率，从而提高吞吐。
- **视觉编码器 / Prefill / Decode 分离部署**：按阶段解耦资源分配，使 TTFT 与 TBT 可以独立控制和优化。

与 DeepSeek-V3 / R1 推理系统一致，Attention 使用数据并行（#DP = #EP）。DP/EP 副本必须做负载均衡，因此系统包含 **Prefill 负载均衡器、Decode 负载均衡器、视觉编码器负载均衡器与专家并行负载均衡器**。为避免重复 Prefill，系统还构建了 **KVCache 存储**。

请求进出时需要在 API 协议与 token 序列之间转换，**DeepSeek Recipe** 负责高效、准确地编解码；它按特殊 token 分段并行编码，既防止特殊 token 注入，又降低长输入的 tokenization 延迟。

参考架构：

![DeepSeek V4.1 Flash 推理系统架构](/picture/deepinfer/architecture_overview.png)

### 整体形态是什么？

分离式的 E/P/D 架构，加上专用存储节点。

### 使用什么并行方式？

- **Prefill**：EP2–8 / DP2–8；在 HBM 允许的前提下，使用尽可能小的 EP/DP。
- **Decode**：EP64–128 / DP64–128（Wide-EP）；在可用 GPU 允许的前提下，使用尽可能大的 EP/DP。
- **视觉编码器（VE）**：每个实例通常一张 GPU 足够，建议使用更便宜的 GPU。
- **目前不使用 TP 与 CP。**

### Wide-EP 为 Decode 带来什么？

Wide-EP 显著降低 MoE 专家部分每步的显存流量；配合良好的 EPLB 支持，可以有效提高 TPS。下图展示 EP8 与 EP64 的基准数据（离线基准脚本，配置见图）。

![EP8 vs EP64 解码吞吐](/picture/deepinfer/wide_ep_decode_pareto.png)

### 多模态如何部署？

采用 EPD 分离，视觉编码器部署在单张 GPU 上。

### 最小部署单元要多大？

1P + 1D 即可部署。实际中 P 与 D 的 GPU 比例大致为 1:1。由于推荐的 Decode 配置是 EP64 或更高，**最小的高效部署单元是 64 张 Prefill GPU 加 64 张 Decode GPU**。

## 三、集群与存储

### 能否跨数据中心部署？

可以。V4.1 Flash 大幅压缩了 KV Cache，因此把 P 与 D 拆到不同数据中心在技术上可行，但会付出可靠性代价（例如跨数据中心光纤容易被施工挖断）。按每 8 张 GPU 约 400 Gbps 互联带宽做预算。实际部署中，DeepSeek 的推理节点位于不同但相邻的数据中心。

### 标准存储服务器配置是什么？

建议每台 KV Cache 存储服务器配 20 块 30.73 TB SSD，每台服务器提供至少 40 GB/s 读、5 GB/s 写吞吐，总容量足以保留 24 小时数据。存储服务器数量按推理部署规模扩展。

### 存储集群与在线 GPU 的比例是多少？

实际按逻辑容量约每 GPU 14.4 TB 计算；物理容量需求更大。

### 3FS 分布式文件系统有什么特点？

它必须在节点故障与网络抖动下同时提供高可用与高性能。数据以 10+2 纠删码存储在 3FS 上，每个 EC 组可同时容忍两个节点故障。一次读取会同时向组内 11 个存储节点发请求，任意 10 个响应即可解码返回。GC Worker 每个存储节点部署一个实例，按 LRU 策略定期回收过期 KV Cache，把磁盘使用率保持在 watermark 以下。

## 四、专家并行负载均衡（EPLB）

### Prefill 的 EPLB 策略是什么？

**机制**：除了自身负责的专家外，每张 GPU 还持有少量冗余专家，作为热点专家的副本。系统收集最近 N 步每个专家的负载，每 M 步根据统计重新分配专家到 GPU；专家权重通过 NVLink 的 `symm_mem` 在组内 shuffle。

**Prefill 设置**：从每 GPU 4 个冗余专家起步（向上取整，使总专家数是 `ep_size` 的倍数），统计窗口 100 步，每 1,000 步重新分配一次。实测 `moe_lb`（最慢 rank / 平均）为 1.074（EP4DP4）/ 1.045（EP2DP2）。

### Decode 的 EPLB 策略是什么？

机制与 Prefill 相同。**设置**：从每 GPU 1 个冗余专家起步（向上取整，使总数是 `ep_size` 的倍数）——例如在 EP64 下，排除共享专家后，取整后约为每 GPU 7 个专家；统计窗口 1,000 步，每 10,000 步重新分配一次。实测 `moe_lb` 为 1.090。

## 五、数据并行负载均衡

### Prefill 负载均衡器如何实现？

**分桶**：按总上下文长度把请求分为 4 档：<16K / 16–64K / 64–256K / ≥256K。根据上一轮各档的计算配额需求占比，动态重新分配各档的引擎数量；每档有自己的单批计算配额上限。

**跨 DP 组**：各引擎按剩余计算配额拉取工作。由于任何引擎都能命中任何前缀，不做前缀亲和路由。

**DP 组内**：一批请求交给某个引擎后，每个请求在到达时被分配到当前配额负载最低的 DP rank。

**单 GPU 内**：请求按 FIFO 出队，chunk size 不固定，而是从预算中求解。每个 Prefill 步给定时间预算（`compute_quota_limit`，单位纳秒），拟合的二次代价模型 `cost(q, c)` 预测该步耗时（`q` 为序列 token 数，`c` 为已计算的前缀长度）。调度器对该二次式求解 `q`，在不超过 32,768 token 预算的前提下取满足时间预算的最大 `q`。预算本身还会在 DP 组内通过 all-reduce MIN 进一步收紧。

### Decode 负载均衡器如何实现？

Decode 负载均衡器在所有 Decode 实例的所有 GPU 上均匀路由。原则是同时平衡两个维度：请求数（决定每步 batch size）与总 KV Cache 长度（决定 Attention 部分成本）。启发式做法是：把短请求发给并发低但总 KV 长度大的 GPU，把长请求发给总 KV 长度小的 GPU。组内调度算法见 DualPath。

### 视觉编码器负载均衡器如何实现？

按机器粒度路由图像：Prefill 机器随机选择一台每 GPU 部署一个视觉编码器的机器，在该机器上把图像交给当前处理图像 patch 数最少的 GPU。patch 数直接由尺寸计算：`(width/patch_size) × (height/patch_size)`。

### 负载均衡整体带来多少提升？

可以与离线基准对比：系统大约达到「完美均衡离线基准」的 80%。

## 六、KV Cache 存储

| 内容 | 介质 | 副本 | 管理方 |
|------|------|------|--------|
| 在途 block KV Cache + state KV Cache | 推理 GPU 显存 | — | Prefill / Decode 实例 |
| SWA KV Cache | 本地主机内存 | 单副本；不写 3FS | Prefill / Decode 实例；跨机命中通过 400G VLAN 拉取 |
| Paged block KV Cache | 专用 3FS 存储集群 | 无应用层副本；依赖 3FS 自身冗余 | `kvcache-store` |

### Block KV Cache

**存储介质是什么？** 主 KV 与 indexer K 组成的 block KV Cache 存放在专用 3FS 存储集群，而不是本地主机内存。与分层存储系统相比，专用存储集群让每台 Prefill 机器都能命中任意前缀，避免前缀亲和与负载均衡路由策略冲突；同时让存储节点扩缩容与计算节点解耦，并使 KV Cache 不受计算节点故障影响（计算节点故障率比存储节点高一个数量级以上）。

**Prefill 是否使用分页 KV Cache？** 是，Prefill 侧也使用分页 KV Cache。

**Decode 时是否开启 KV offload？** 不开启。Decode 不受缓存容量约束，因此也不使用 Hi-Sparse。Decode 阶段产生的 KV Cache 仍会写入磁盘，以提高缓存命中率。

**过期与 TTL？** `kvcache-store` 侧没有过期逻辑；回收完全由 3FS 自身的 GC 完成。3FS 为 KV Cache 提供约 100 PiB SSD 存储；Agent 流量的 TTL 为 3 天，Chatbot 流量为 7 天。Chatbot 流量的命中长尾更重：少量 KV Cache 被非常多次读取，丢失它的整体命中率代价明显。经验上，约 3 天的 TTL 会让 Chatbot 流量损失 0.5%–1% 的 KV Cache 命中率。

**缓存相关开销如何被掩盖？** 匹配由 Entrypoint 对本地 `kvcache-store` 以 CHECK 角色完成，只对元数据做哈希链前缀匹配；但命中长度必须在路由前算出，因此这部分不能隐藏。**拉取 KV** 在独立进程中异步进行，把 KV 从 3FS 搬到主机共享内存。**Swap-in** 从共享内存到 HBM 在引擎进程内的线程中运行，目标是让逐层 H2D 拷贝与前向计算重叠。由于 CUDA copy engine 对细粒度 KV Cache block 拷贝表现不佳，swap-in 使用 GDR 实现：RDMA 网卡可以在不消耗 SM 的情况下打满 PCIe 带宽。

### SWA KV Cache

**如何存储？** SWA KV Cache 不进入 paged KV block，而是位于 state cache 中，每个序列占一个固定大小的槽（`StateBlock`）。每个槽按是否需要存储分为两个区域：`[stored region: SWA] [dropped region: YOCO consumer SWA]`。存储区是需要跨请求保留的 SWA 窗口；丢弃区存放 YOCO consumer SWA 层（其恢复依赖 Prefill 重算）以及 C2 压缩边界累加器 `ca_state` / `ica_state`（压缩窗口边界天然对齐 128 token 存储粒度，因此不存在跨边界依赖）。对 V4.1，存储层是前 20 层。

SWA KV Cache 只在 128 token 的整数倍位置存储。Prefill 后，存储位置通过预留序列末尾一个 token 并向下取整到 128 的倍数得到；如果剩余不足一个完整 128-token 单元，本轮不存。分块 Prefill 过程中不存中间结果，只有最后一个 chunk 存。Decode 阶段在设备显存中分配独立副本区域；请求每跨过 128 token 边界，就把当前窗口在设备内复制到副本区。请求结束时，从副本区最近一次完成的拷贝取存储内容，附件点向下取整到 128 边界。拷贝位置最多超过边界 5 个 token，由槽 ring buffer 比窗口多出的 6 个位置吸收。

**存什么、存在哪？** 每个序列只有 `state_size_to_disk` = 20 层 × 134 token × 528 B = **1,415,040 B** 进入存储。528 B/token 是 MXFP8 SWA KV 格式。SWA KV Cache 不进入持久缓存，而是留在产生它的机器内存中。一个内存目录（逻辑统一，但物理上按 key 分区到各 Entrypoint 机器）记录哪台机器持有它；未命中时，没有本地副本的机器通过远程拉取获取。

**如何从 Prefill 传到 Decode？** SWA KV Cache 与 block KV Cache 走同一条 P→D 传输路径。

**支持跨机命中吗？** 支持。跨机命中依赖 Entrypoint 的内存目录，以及 400G VLAN 回拉。

### KV Cache 命中与重算

**重算如何做？** SWA bounded replay 只重算最近 128 token，并把注意力窗口截断到重放段内：从位置 `s` 重放时，位置 `i` 的 query 只与 `max(s, i−127)` 到 `i` 的 key 计算，从而得到近似的 SWA KV Cache。重算成本因此降为常数，与上下文长度和层数无关。该机制用于两处：

- **Encoder SWA bounded replay**：前缀缓存命中长度只取决于 block KV Cache。SWA KV Cache 缺失时，重算命中前缀的最后 128 token，并与未缓存后缀一起前向；重放段只重建 SWA KV Cache，block KV Cache 直接复用，既不重算也不覆盖。
- **Decoder SWA bounded replay**：Prefill 计算只发生在 encoder 层。CED 架构中 Decoder 层的 block KV Cache 由 encoder 最后一层隐状态投影而来，因此只需逐层计算 Decoder SWA KV Cache；若要精确重建，需要把最后 2,560 token 跑过 20 个 Decoder 层。实际每次 Prefill 只重算 prompt 最后 128 token 的 Decoder 层，结果只用于 Decode，不进入前缀缓存——这几乎把总 Prefill 计算减半。

**命中策略是什么？** 原则是：命中长度由 block KV Cache 决定，最后一个 block 是否需要重算由 SWA KV Cache 决定——SWA 未命中就回退一个 block 并重算。具体地，在 YOCO 截断要求下，长度 `L` 的输入最大命中长度为 `L − 134`（128-token 窗口 + 6 token 投机回溯）。在该上限下先查 block KV Cache，得到命中长度 `L1`（128 token block 的倍数），再查 SWA KV Cache；如果 `L1 − 128` 到 `L1` 范围内存在 SWA KV Cache，命中长度为 `L1`，否则为 `L1 − 128`，其中 128 是重算窗口。

> ### 💡 新人小结（编者注）：KV Cache 不是「一个缓存」，而是三层
>
> - **GPU 显存**：在途 block KV 与 state KV，最快但容量最小。
> - **本地主机内存**：SWA KV，单副本，跨机命中走 400G VLAN。
> - **3FS 存储集群**：paged block KV，容量约 100 PiB，命中任意前缀，不依赖计算节点。
> - 三层各管一段，缺一层就会在长上下文或高并发下撞到不同的墙。

## 七、自动扩缩容与容错

### 自动扩缩容如何实现？

Prefill 根据非缓存 token 数决定所需机器数；Decode 根据运行中的请求数调整。调整有速率限制，并有上下界。

### 冗余配额是什么样？

部署会保留冗余配额以吸收流量波动，有两种形式：池中保留未占用的就绪机器作为扩容库存；以及始终让在线实例数多于计算出的需求——具体来说，配额利用率高于 0.8 时禁止缩容。

### 重启需要多久？

重启在分钟级。容错来自 PE（Prefill 引擎）/ CE（Decode 引擎）的配额冗余 + 请求级重路由，而不是缩短单个引擎的重启时间。

### EP 故障转移策略是什么？

在大 EP 部署中，一个实例的所有 GPU 构成一个通信域，因此单张 GPU 故障会让整个实例不可用。重启时，为了让实例剩余健康 GPU 继续服务，平台会根据当前所有健康 GPU 重新确定 EP size，实例以实际拿到的 GPU 数作为 EP size。

## 八、低精度计算

### FP4 用在哪里？

- **主 KV**：per-16，e4m3——类似 NVFP4，但没有全局 scale。
- **Indexer KV**：per-32，ue8m0，MXFP4。
- **MoE routed-expert 权重**：per-32，ue8m0，MXFP4。

## 九、内核实现

### 使用哪些内核（内核后端）？

用仓库中的 `examples/benchmark.py` 做 profile，得到对应 trace，其中包含内核使用信息。

### 什么时候使用 Mega MoE？

Mega MoE 是 scale-up 域内的通用方案：无论 EP size 或 batch size 如何，Mega MoE 都明显快于未融合的 MoE。

### DeepGEMM / FlashMLA 等内核库有更好的内部版本吗？

大体相同；indexer 与 mega 系列略有差异。FlashMLA 缺少一项手工调优的 SASS 优化，使 Decode 内核慢 5%–10%。端到端性能接近，最多相差 5%。

## 十、投机解码

### 如何实现？

在推理系统层面，DSpark 使用变长验证。由于草稿 token 的接受概率随请求与位置变化，固定长度验证会在低概率 token 上浪费验证算力。DSpark 用 Confidence Head 预测每个草稿 token 的接受率，并在调度时把验证算力分配给边际收益最高的 token。生产中，草稿模型一次生成 5 个 token；根据 profile 出的硬件吞吐曲线，以及每个请求、每个草稿 token 的接受率，对草稿 token 按预期接受率排序，并选择使有效吞吐最大的前缀子集进行验证。

### 变长验证相对固定 5 个草稿 token 提升多少吞吐？

变长验证把每步验证的草稿 token 数减少约 30%（5.00 → 3.2–4.4），平均接受长度基本不变（3.29 vs 3.20）。根据 EP64/DP64 基准数据计算，在生产工作点（每 rank 约 60 个并发序列）上带来 **+4%–7%** 的吞吐提升。

| 并发（seqs/rank） | 固定 5 个草稿的平均验证长度 | 变长验证的平均验证长度 | 固定 5 个草稿的接受长度 | 变长验证的接受长度 | 固定 5 个草稿的估算吞吐（tok/s/rank） | 变长验证的估算吞吐（tok/s/rank） | 变长相对固定 5 个的收益 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 16 | 5 | 4.44 | 3.32 | 3.24 | 4891 | 4816 | −1.5% |
| 32 | 5 | 3.65 | 3.25 | 3.17 | 8964 | 9111 | +1.6% |
| 48 | 5 | 3.94 | 3.3 | 3.26 | 12032 | 12467 | +3.6% |
| 60 | 5 | 3.58 | 3.3 | 3.21 | 13730 | 14466 | +5.4% |
| 72 | 5 | 3.24 | 3.29 | 3.12 | 14962 | 15961 | +6.7% |
| 96 | 5 | 3.24 | 3.3 | 3.15 | 16817 | 18660 | +11.0% |

### 有更好的内部草稿模型版本吗？

内部版本略新，但性能差异很小。

### 它如何与异步调度交互？

`schedule(N+1)` 运行时，第 N 步的置信度尚未返回，因此 CPU 调度只固定总预算；每请求长度由 GPU 上的 TileLang 内核重新计算（累积乘积、top-k 全局排序，配合融合的 remap scatter），结果写回 pinned buffer。

## 十一、异步调度（零开销批调度器）

### 优化的是什么？

Decode 第 N 步执行 `schedule(N) → forward(N) → sample(N) → post-process(N)`。其中 schedule 与 post-process 是纯 CPU 逻辑，并且在 `post-process(N)` 前需要一次 GPU-CPU 同步来取得 `sample(N)` 的结果。因此 GPU 在 `post-process(N)` 与 `schedule(N+1)` 期间空闲。异步调度的目标是消除这段空闲。

### 如何实现？

沿用现有实现（SGLang、NanoFlow）的思路：在 GPU 与 CPU 之间异步调度，让流水线重叠。GPU 执行 `forward(N) → sample(N)` 时，CPU 并发执行 `schedule(N+1) → launch_forward(N+1) → post-process(N−1) → launch_sample(N+1)`，使 GPU 全程没有空闲。

## 十二、结构化输出

### 什么是结构化输出？

结构化输出（SO）把模型输出严格约束为指定格式（JSON Schema、正则表达式、EBNF 等）。

### 工作流程是什么？

编译与推理解耦：SO 编译异步执行，请求在进入引擎前取得编译结果。

- **编译阶段**（在专用 SO 机器上）：把格式约束编译成语法自动机。
- **运行时**：每个 Decode 步推进自动机，确定合法 token 集合，生成 mask 并应用到采样逻辑，强制输出符合目标结构。

## 十三、DeepSeek Recipe

### 多协议 API 支持如何实现？

DeepSeek Recipe 把 Chat Completions API、Responses API 与 Messages API 的请求转换成统一的会话格式，共享一套模型模板与编码逻辑，为给定模型版本生成 prompt 与 token 序列。

模型输出被解析为事件——正文、推理内容、工具调用等——再转换为各协议的流式响应。请求转换、prompt 编码与输出解析都用 Rust 实现以获得高效处理，并提供 Python 绑定以便使用。

### 分词及其加速如何处理？

**输入编码**：构建 prompt 时，显式插入的特殊 token 及其位置会被记录，因此编码时直接映射到对应 token ID；普通文本用基础词表编码。用户输入、工具结果或工具调用参数中出现的 `<｜User｜>`、`<｜image｜>` 等字符串，会按普通 token 序列编码，防止文本注入控制 token 影响模型行为。

**输出解析**：处理响应 token 流时，按 token ID 识别推理结束标记 `</think>` 与 DSML 标签中的 `｜DSML｜` 特殊 token。模型用普通 token 拼出的同样文本会被正确保留为字面内容，不会获得触发推理结束或工具调用的能力。

**分段加速**：构建 prompt 时，特殊 token 的位置与边界已知。根据 BPE 分词器的工作方式，特殊 token 之间的普通文本段可以独立编码。因此跳过对普通文本内部特殊 token 的匹配与切分，并行编码各段，再按原顺序拼接。这降低了超长上下文请求的分词时间，并改善请求延迟。

### 流式输出如何解析？

推理后端按 token 粒度返回 chunk，因此单个 UTF-8 字符、一个 DSML 标签或一个工具参数可能跨多个 chunk。流式解析需要一个状态机，保留进行中的状态并及时返回已确定内容，以降低响应延迟。

**跨 chunk 解析**：解码遇到不完整 UTF-8 字符时，缓存相关 token，直到后续 token 补齐后再交给解析器。解析器在推理、正文与工具参数阶段之间维护状态，并对分隔标记使用增量 KMP 匹配。如果 chunk 在标记中间结束，保留匹配进度，下一 chunk 到达时继续识别；已确定内容随解析推进即时输出，无需反复扫描前文。

**工具参数转换**：工具名一解析出来就可以返回，之后直接从 DSML 参数边界生成 JSON 片段。字符串值逐段做 JSON 转义，数组与对象等非字符串值按其原始 JSON 文本输出，不需要构造完整参数对象，也不需要反复解析与序列化。即使是长参数，也可以边生成边返回片段，缩短首段参数等待时间。

> ### 💡 新人小结（编者注）：这十二节其实在回答四个问题
>
> - **怎么切**：E/P/D 分离、Attention DP、MoE EP/Wide-EP、EPLB。
> - **怎么省**：三层 KV Cache、前缀复用、SWA bounded replay、低精度。
> - **怎么快**：DSpark 变长验证、异步调度、Mega MoE、分段分词。
> - **怎么稳**：两级负载均衡、配额冗余、分钟级重启、EP 故障重算。
> - 四个问题都围绕同一目标：在给定硬件与延迟要求下，提高有效吞吐。

## 要点总结

1. **这份问答描述的是生产系统**：155 万亿 token/天、每 GPU 小时 38 美元等数字来自 DeepSeek 自有集群、特定流量与谷时电价，不是通用基准。
2. **架构核心是分离与并行**：VE/Prefill/Decode 分离，Attention DP、MoE EP/Wide-EP、EPLB 与两级负载均衡共同工作。
3. **KV Cache 分三层**：GPU 显存、本地主机内存、3FS 存储集群；block KV 在 3FS，SWA KV 留在产生它的机器内存。
4. **性能优化是条件性的**：变长验证在生产工作点 +4%–7%，低并发时可能为负；负载均衡只达到完美均衡离线基准的约 80%。
5. **开源边界明确**：KV Cache 管理与负载均衡组件未完全开源，完整在线服务需要自建或接入相应基础设施。
6. **Recipe 负责请求边界**：多协议转换、分词、流式解析与结构化输出约束，逻辑与官方 API 对齐。

> ### 💡 新人小结（编者注）：读完之后该带走什么
>
> - 不要把问答里的生产数字直接写进自己的容量模型；先确认硬件、模型与流量结构是否可比。
> - 不要只抄并行参数；EP/DP 的选择受 HBM、超节点规模与带宽约束。
> - 不要忽略未开源部分；KV Cache、阶段拆分与跨副本负载均衡是完整服务的一半工程量。
> - 真要用 DeepInfer，先按 README 的硬件与构建条件做一次离线推理，再谈在线服务。

## 原文链接

- [DeepSeek V4.1 Flash Inference Deployment Q&A（英文原文）](https://github.com/deepseek-ai/DeepInfer/blob/main/doc/inference_deployment_qa.md)
- [DeepInfer 仓库](https://github.com/deepseek-ai/DeepInfer)
- [DeepSeek V4.1 Flash 推理系统架构（知乎长文）](https://zhuanlan.zhihu.com/p/2095560295930508953)
- [Apache License 2.0](https://github.com/deepseek-ai/DeepInfer/blob/main/LICENSE)
