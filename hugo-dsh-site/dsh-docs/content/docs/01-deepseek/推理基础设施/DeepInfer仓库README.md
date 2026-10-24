---
title: DeepInfer 仓库 README（中文记录）
description: DeepInfer 官方 README 的中文记录：定位、支持模型与硬件、功能与配置、构建与离线推理、性能基准、许可证与致谢。
weight: 2
---

本文是 DeepInfer 官方仓库 `README.md` 的中文记录，按原文结构逐节翻译，供离线阅读与引用。命令、环境变量、参数名、数字与专有名词保留原文写法；如译文与英文原文有冲突，以[英文原文](https://github.com/deepseek-ai/DeepInfer/blob/main/README.md)为准。

> 记录时点：2026-10-24。仓库最近一次推送为 2026-10-21。原文采用 Apache 2.0 许可；本文作为中文记录同样遵循该许可，并保留原文链接与仓库出处。

## 为什么需要这份记录？（编者说明）

DeepInfer 的一手信息分散在 README、部署问答与代码注释里，且 README 是英文、会随仓库更新。把 README 单独落成一篇中文记录，解决三个实际问题：

- **离线可读**：站点可以完全离线打开，不必每次都访问 GitHub。
- **引用可追溯**：站内文章提到「README 说支持哪些硬件」「README 说缺哪些组件」时，可以直接指到这一篇，而不是只给一个外链。
- **口径可对照**：命令、环境变量与默认值保留英文原样，读者复制时不需要二次翻译。

| 项 | 内容 |
|----|------|
| 原文 | `deepseek-ai/DeepInfer` 仓库根目录 `README.md` |
| 记录范围 | 概述、支持模型、功能与配置、硬件、构建、离线推理、基准、许可证、致谢、引用 |
| 记录时点 | 2026-10-24（仓库最近推送 2026-10-21） |
| 许可 | Apache License 2.0 |
| 翻译原则 | 结构按原文；命令/变量/数字不改写；冲突时以英文原文为准 |

> ### 💡 新人小结（编者注）：这份 README 只回答三件事
>
> - **能不能跑**：支持哪款模型、哪两类加速卡、需要什么编译环境。
> - **跑起来有什么**：并行、投机解码、SWA replay、异步调度、结构化输出等能力清单。
> - **还缺什么**：完整在线服务所需的 KV Cache、阶段拆分与跨副本负载均衡，本次没有一并开源。
> - 把这三件事记住，就不会把「开源引擎」误读成「开箱即用的云服务」。

---

## 一、概述

DeepInfer 是支撑 DeepSeek 大规模生产服务的高性能推理框架。本次开源版本提供**离线推理**，包含大规模服务所需的核心能力。

## 二、支持的模型

- DeepSeek-V4.1-Flash

## 三、支持的功能

- 优化后的内核、模型与采样器运行时
- 并行：Attention DP / 均衡 CP，MoE EP + EPLB
- DSpark 与置信度调度验证
- SWA bounded replay
- 异步调度（零开销批调度器）
- Batch-invariant 推理
- 确定性采样
- 结构化输出

## 四、功能细节与配置

以下内容对应原文折叠区「Feature details and configuration」。

### 优化内核

集成以下内核与算子库：

- [FlashMLA](https://github.com/deepseek-ai/FlashMLA)
- [DeepGEMM](https://github.com/deepseek-ai/DeepGEMM)
- [DeepEP](https://github.com/deepseek-ai/DeepEP)
- [TileKernels](https://github.com/deepseek-ai/TileKernels)
- [DeepSelect](https://github.com/deepseek-ai/DeepSelect)
- 以及位于 `deepinfer/model_executor/layers/tilelang_ops` 的自定义 [TileLang](https://github.com/tile-ai/tilelang) 内核

### DSpark 与置信度调度验证

用草稿 token 加速解码，并用置信度估计来安排验证。启用方式：

```bash
export DEEPINFER_ENABLE_MTP_VARLEN=1
```

`DEEPINFER_USE_NEXTN_DECODE` 控制最大草稿长度，默认值为 **5**。

### Decoder 侧 SWA bounded replay

只重放一个有界后缀，以减少 Decoder 的预填充工作量。V4.1 的默认设置为：

```bash
export DEEPINFER_YOCO_PREFILL_TRUNCATION_MODE=compact
export DEEPINFER_YOCO_PREFILL_KEEP_TOKENS=140
```

保留的后缀必须覆盖 SWA 窗口加上最大回溯长度。

### 异步调度

让 CPU 的批调度与设备执行重叠。默认开启。

### Batch-invariant 推理

在不同批次组成下保持模型结果一致。启用方式：

```bash
export DEEPINFER_BATCH_INVARIANT=1
```

该能力**仅支持 NVIDIA**。

### 确定性采样

对于相同的输入 token 与采样参数，产生相同的输出。启用方式：

```bash
export DEEPINFER_BATCH_INVARIANT=1
export DEEPINFER_ENABLE_MTP_VARLEN=0
```

还需要向 `SamplingParams` 传入固定的 `seq_seed`。该能力**仅支持 NVIDIA**。

### 结构化输出

把生成内容约束为正则表达式、JSON Schema 或 EBNF 语法。向 `SamplingParams` 传入 `so_args={"schema_type": ..., "schema": ...}`；示例见 `examples/_structured_output.py`。

## 五、支持的硬件

- NVIDIA SM100 GPU
- 华为昇腾 950 NPU

## 六、完整推理服务还需要什么（本次未包含）

- KV Cache 系统
- VE / Prefill / Decode 拆分部署（大部分所需的引擎功能已经实现）
- Prefill 与 Decode 两侧跨数据并行副本的负载均衡

> ### 💡 新人小结（编者注）：功能清单要分三层读
>
> - **引擎层能力**：并行、投机解码、SWA replay、异步调度、结构化输出——这些在开源代码里。
> - **部署层能力**：KV Cache 系统、阶段拆分、跨副本负载均衡——README 明确说本次未包含。
> - **硬件层前提**：SM100 或昇腾 950，加上 Linux、GCC 13、Python 3.12、CUDA 13/CANN 9.2。
> - 「功能支持」不等于「服务完整」，三层要分开核对。

## 七、快速开始

### 前置条件

- Linux
- GCC >= 13.0
- Python 3.12
- CUDA 13（仅 NVIDIA）
- CANN 9.2（仅昇腾）

### 从源码构建

构建前请先创建并激活一个空的 Python 3.12 环境。

```bash
# 昇腾平台把 nvidia 改为 ascend
# 需要在运行 DeepInfer 的环境中持续导出
export DEEPINFER_ACCELERATOR=nvidia
./build.sh
```

### 离线推理

默认 checkpoint 为 [`deepseek-ai/DeepSeek-V4.1-Flash-1017`](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017)，首次运行时下载到 Hugging Face 缓存；也可以设置本地目录：

```bash
export DEEPINFER_HF_MODEL_DIR=/path/to/DeepSeek-V4.1-Flash-1017
python examples/offline_inference.py
```

更多示例见 `examples/README.md`。

## 八、性能基准

原文给出两张 Pareto 图：Prefill 的「非缓存输入吞吐 vs 延迟」，以及 Decode 的「输出吞吐 vs 单用户速度」。

![Prefill：非缓存输入吞吐 vs 延迟（4 张 SM100）](/picture/deepinfer/prefill_pareto.png)

![Decode：输出吞吐 vs 单用户速度（64 张 SM100）](/picture/deepinfer/decode_pareto.png)

测量环境：Prefill 使用 4 张 SM100 GPU，Decode 使用 64 张 SM100 GPU，各自位于同一个 scale-up 域内。复现方式见 `examples/sweep/run_sweep.sh`。

> README 只给出自身吞吐-延迟曲线，没有提供与其他推理引擎的同条件对比，也没有给出通用加速倍数。引用这些图时，应同时写明硬件、模型与测量口径。

## 九、评测基准

见 `eval/README.md`。

## 十、许可证

DeepInfer 以 **Apache License 2.0** 发布，详见仓库 `LICENSE`。

## 十一、致谢

DeepInfer 最初主要基于 [vLLM](https://github.com/vllm-project/vllm) 构建，并借鉴了 [SGLang](https://github.com/sgl-project/sglang) 与 [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) 的设计。原文对上述项目的开发者致谢。

## 十二、引用

原文给出的 BibTeX：

```bibtex
@misc{deepinfer2026,
  title = {DeepInfer: High-performance, large-scale inference framework for DeepSeek models},
  author = {DeepSeek-AI},
  year = {2026},
  publisher = {GitHub},
  howpublished = {\url{https://github.com/deepseek-ai/DeepInfer}},
}
```

## 附：术语对照

| 术语 | 中文 | 在本文中的含义 |
|------|------|----------------|
| DeepInfer | — | 本文记录的推理框架 |
| Prefill | 预填充 | 处理输入 prompt、建立 KV Cache 的阶段 |
| Decode | 解码 | 逐 token 生成输出的阶段 |
| KV Cache | KV 缓存 | 缓存历史 token 的 Key/Value，避免重复计算 |
| EP | 专家并行 | 把 MoE 专家分布到不同 GPU |
| EPLB | 专家并行负载均衡 | 按负载重新分配专家，缓解热点 |
| Wide-EP | 宽专家并行 | Decode 侧使用较大的 EP 规模 |
| DP / CP / TP | 数据 / 上下文 / 张量并行 | 三种不同的切分方式 |
| DSpark | — | DeepSeek 的投机解码方案 |
| SWA bounded replay | 滑动窗口有界重放 | 只重算最近窗口，降低长上下文成本 |
| Batch-invariant | 批次不变 | 不同批次组成下结果保持一致 |
| Recipe | — | deepseek-recipe，负责协议转换与分词 |

## 附：部署前检查清单

| 检查项 | 要求 | 不满足的后果 |
|--------|------|--------------|
| 模型 | DeepSeek-V4.1-Flash | 当前公开版不支持其他模型 |
| 加速卡 | NVIDIA SM100 或昇腾 950 | 其他硬件未被列入支持范围 |
| 驱动与工具链 | CUDA 13 或 CANN 9.2 | 无法按 README 构建或运行 |
| 编译环境 | Linux、GCC >= 13、Python 3.12 | 构建脚本无法执行 |
| Python 环境 | 空的 3.12 环境 | 依赖冲突可能覆盖现有项目 |
| 加速平台变量 | `DEEPINFER_ACCELERATOR=nvidia` 或 `ascend` | 构建目标不确定 |
| 权重 | 默认 HF checkpoint 或本地目录 | 首次运行会尝试下载；离线环境需本地路径 |
| 服务形态 | 先按离线推理验证 | 在线服务还需 KV Cache、阶段拆分与负载均衡 |
| 合规 | 保留 Apache 2.0 许可与 NOTICE | 再分发时可能违反许可要求 |

## 与站内其它文章的关系

- 想看「DeepInfer 为什么重要、与权重是什么关系」：读[DeepInfer 推理框架](/docs/01-deepseek/推理基础设施/deepinfer推理框架/)。
- 想看「生产系统为什么这样设计、数字在什么条件下成立」：读[推理部署问答（中文记录）](/docs/01-deepseek/推理基础设施/deepseek-v4.1-flash推理部署问答/)。
- 想核对 V4.1-Flash 的发布时间与参数口径：[模型发布时间线](/docs/01-deepseek/模型全景/模型发布时间线/)、[模型参数对比](/docs/01-deepseek/模型全景/模型参数对比/)。

## 记录要点

1. DeepInfer 是推理框架，本次开源版本面向 **DeepSeek-V4.1-Flash 的离线推理**。
2. 支持硬件只有 NVIDIA SM100 与华为昇腾 950，环境要求 Linux、GCC 13、Python 3.12、CUDA 13/CANN 9.2。
3. 引擎核心能力包括并行、DSpark、SWA replay、异步调度、Batch-invariant、确定性采样与结构化输出。
4. 完整在线服务仍缺 KV Cache 系统、VE/Prefill/Decode 拆分部署与跨 DP 副本负载均衡。
5. 性能图只说明自身在 4 卡 Prefill / 64 卡 Decode 下的吞吐-延迟取舍，不能当作跨引擎的通用倍数。

> ### 💡 新人小结（编者注）：README 的正确用法
>
> - 先看「支持模型 + 硬件」，确认自己是否在范围内；不在范围内，后面的性能数字与你无关。
> - 再看「功能 + 未包含」，区分引擎能力与完整服务能力。
> - 最后看「构建 + 示例」，按 README 的环境要求准备，不要拿 A100/H100 的经验直接套 SM100。

## 原文链接

- [DeepInfer README（英文原文）](https://github.com/deepseek-ai/DeepInfer/blob/main/README.md)
- [DeepInfer 仓库](https://github.com/deepseek-ai/DeepInfer)
- [Apache License 2.0](https://github.com/deepseek-ai/DeepInfer/blob/main/LICENSE)
