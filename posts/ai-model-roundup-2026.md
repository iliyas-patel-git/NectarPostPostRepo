---
title: "The 2026 Open-Weight LLM Roundup: Architectures & Deployment"
slug: "ai-model-roundup-2026"
date: "2026-02-14"
updated: "2026-10-14"
hub: "ai"
tags: ["roundup","llm","models","open-source","edge-ai","gemma-4","qwen3.5","qwen3.6","mellum"]
summary: "A dated, structural comparison of the small open-weight language models that matter in 2026, from Gemma 4 and Qwen3.5 to Ministral 3 and JetBrains Mellum, plus a practical guide to quantization and edge or browser deployment."
readTime: "9 min read"
coverAccent: "from-indigo-900 via-slate-900 to-cyan-950"
---

> **Updated October 14, 2026.** First published February 14. Since then Google shipped Gemma 4 (including a 12B Unified model in June), Alibaba released the Qwen3.5 small series and Qwen3.6, and several large open-weight models landed this month. Every model below now carries a release date so you can judge how current it is.

## The Rise of Compact Reasoning Models

The race in AI has moved away from building ever-larger models and toward packing strong reasoning into small ones. Models between 1B and 12B parameters now handle chat, summarization, tool calling and image or audio input on a laptop, a phone, or inside a browser tab. Most of the leading releases also ship under **Apache 2.0**, which removes much of the legal friction around commercial use.

Four architectural ideas drive most of the progress:

- **Per-Layer Embeddings (PLE):** each decoder layer gets its own small embedding per token, so a model can be large on disk but cheap to run. Gemma 4's "E" (effective) sizes use this.
- **Hybrid linear attention:** layers such as Gated DeltaNet (Qwen3.5) replace some full-attention layers, which keeps memory growth under control at long context.
- **Encoder-free multimodality:** Gemma 4 12B drops the separate vision and audio encoders and feeds inputs straight into one decoder stack.
- **Multi-token prediction (MTP):** drafter models for speculative decoding give faster inference without changing outputs. Google published Gemma 4 MTP drafters on April 16, 2026.

## Release Timeline (2026)

| Date | Release | Why it matters |
| ---- | ------- | -------------- |
| Feb 17 | Qwen3.5 flagship family | Foundation for the small series below |
| Feb 24 | Qwen3.5 medium (27B, 35B-A3B, 122B-A10B) | Mid-size tier |
| Mar 2 | **Qwen3.5 small: 0.8B, 2B, 4B, 9B** | Native multimodal, Apache 2.0, edge-ready |
| Mar (mid) | Mistral Small 4 | Mixture-of-Experts, merges instruct, reasoning and agentic skills |
| Mar 31 / Apr 2 | **Gemma 4: E2B, E4B, 26B A4B, 31B** | Apache 2.0, audio on edge sizes. Google's release notes log March 31, and the public launch coverage is dated April 2 |
| Apr 16 | Gemma 4 MTP drafters; Qwen3.6-35B-A3B | Faster inference; agentic coding MoE |
| Apr 22 | Qwen3.6-27B | Dense 27B tuned for coding |
| Jun 3 | **Gemma 4 12B Unified** | Encoder-free, text, image, audio and video, 256K context, laptop-sized |
| Aug 3 | Qwen3.8-Max | Large flagship; Alibaba says weights will follow |
| Aug 28 | Qwen3.8-Flash-Next | Alibaba calls it an early preview of the Qwen4 architecture |
| Oct 5 | Reflection **Beam** (501B MoE, 23B active) | Weights promised this month under Apache 2.0 |
| Oct 6 | Mistral **Large 4** preview | About 1T parameters; weights promised by end of October |
| Oct 8 | JetBrains **Mellum2.1 Thinking** (12B total, 2.5B active) | Small open-weight coding model for agents |

## Notable Small Models

### 1. Gemma 4 (Google DeepMind): March 31 / April 2, 2026

Gemma 4 launched in four sizes: **E2B**, **E4B**, a **26B A4B Mixture-of-Experts**, and a **31B dense** model. It is released under **Apache 2.0**, a change from the custom Gemma Terms of Use on earlier generations.

- All models accept text, image and video input; E2B and E4B add **native audio input**.
- Context is 128K on the edge models and 256K on the larger ones.
- **Gemma 4 12B Unified (June 3, 2026)** fills the gap between E4B and 26B. It has about 11.95B parameters, 48 layers, sliding-window attention, a 256K context window and no separate vision or audio encoders. Reports say it fits on a 16 GB laptop and reaches about 21 tokens per second on an RTX 4060 with quantized weights.

**Best for:** phones and embedded devices (E2B/E4B), and local multimodal assistants on a laptop (12B).

### 2. Qwen3.5 Small Series (Alibaba): March 2, 2026

Four models, **0.8B, 2B, 4B and 9B**, each with a base version. They share a Gated DeltaNet hybrid architecture, are natively multimodal, and support a native context of about 262K tokens. Apache 2.0 licensed, and available through Ollama.

- **0.8B / 2B:** phone, Raspberry Pi and IoT territory.
- **4B:** a strong base for lightweight agents, and small enough for an entry-level laptop or a browser.
- **9B:** Alibaba reports it beating the previous-generation Qwen3-30B-A3B on several benchmarks.

Reported weak spot: the 4B and 9B trail on pure math and hard reasoning compared with document and vision tasks, so test your own workload.

**Best for:** multilingual products, edge agents, and families where you want one architecture across several sizes.

**What came after:** Alibaba has since released Qwen3.6 (35B-A3B on April 16, 27B on April 22) and the proprietary Qwen3.7 and Qwen3.8-Max models. I did not find new sub-9B open Qwen models after the 3.5 small series. Alibaba said on September 22, 2026 that Qwen4 is in training, with no release date announced.

### 3. Ministral 3 (Mistral AI): December 2025

Ministral 3 ships in **3B, 8B and 14B** sizes, each with base, instruct and reasoning variants. The 3B pairs a 3.4B language model with a 0.4B vision encoder and fits in roughly 8 GB of VRAM at FP8, or less when quantized further. Apache 2.0 licensed. Mistral Small 4 (March 2026) is the larger, MoE-based sibling.

**Best for:** European data-residency projects and vision-plus-text edge workloads.

### 4. JetBrains Mellum2.1 Thinking: October 8, 2026

A 12B-total, 2.5B-active Mixture-of-Experts model aimed at coding agents and fast sub-agents. Active compute is small, but you still need memory for all 12B weights. I have not benchmarked it myself, so compare it against Qwen3.6 and Gemma 4 on your own code before switching.

**Best for:** coding agents and fast sub-agent calls, the slot where I originally recommended Qwen 2.5 Coder 1.5B.

### 5. Phi-4-mini (Microsoft): February 2025

A 3.8B model with a 128K context window under the **MIT license**. As of September 2026, one comparison notes that Microsoft had not shipped a Phi-5 or Phi-4.1 successor, so this remains the current small Phi.

**Best for:** math-heavy and logic-heavy tasks on small machines.

### 6. SmolLM3-3B (Hugging Face): July 2025

A fully open 3B instruct and reasoning model, with the training recipe documented alongside the weights. It is the successor to the SmolLM2 line from the original version of this post.

**Best for:** research, reproducibility, and teams that want to know exactly how their model was trained.

### Still relevant: older generations

- **Llama 3.2 1B and 3B (September 25, 2024):** text-only, but still the reference on-device Llama models. One September 2026 comparison notes Meta has not released a small Llama 4 variant under 8B parameters. Strong tooling support through llama.cpp and MLX.
- **Gemma 3 (March 12, 2025) and Gemma 3n (June 26, 2025):** many production apps still run on these, but new projects should start with Gemma 4.
- **Qwen 2.5 Coder 1.5B (September 2024):** superseded for most new work by the models above.

## Quick Selection Guide

| If you need...                           | Start with                       |
| ---------------------------------------- | -------------------------------- |
| Phone or embedded device, offline audio  | Gemma 4 E2B / E4B                |
| Tiny footprint, long context             | Qwen3.5-0.8B / 2B                |
| Laptop or browser agent                  | Qwen3.5-4B                       |
| Best quality around 9B                   | Qwen3.5-9B                       |
| Multimodal on a 16 GB laptop             | Gemma 4 12B Unified              |
| Coding agent with low active compute     | Mellum2.1 Thinking or Qwen3.6    |
| Reasoning on modest hardware             | Phi-4-mini                       |
| Single-GPU vision + text                 | Ministral 3 3B / 8B              |
| Fully reproducible training              | SmolLM3-3B                       |
| Widest ecosystem and tooling support     | Llama 3.2 1B / 3B                |

## Quantization Formats Demystified

- **FP16 / BF16:** full floating point. Best quality, but too heavy for most browser and mobile memory budgets.
- **FP8:** a middle step used by vendors such as Mistral for single-GPU deployment.
- **Q8_0:** 8-bit precision, with virtually no degradation from the original weights at about half the FP16 memory footprint.
- **Q4_K_M:** 4-bit medium quantization, the usual sweet spot between reasoning retention and RAM use. As a rough guide, a 0.8B model lands near 1 GB and a 4B model near 3.4 GB at 4-bit, plus runtime overhead for the KV cache.

**Rule of thumb:** budget the model file size, then add 1 to 2 GB for context and runtime. Long context is where memory surprises happen, which is why the hybrid attention designs above matter in practice.

## Deployment Notes

- **Browser:** WebLLM and WebGPU make 0.8B to 4B models practical in a tab. Download size, not inference speed, is usually the limiting factor, so lazy-load and cache the weights.
- **Mobile and edge:** llama.cpp, MLX, and Google's AI Edge tooling cover most targets.
- **Laptop or server:** Ollama gets you running in one command (for example `ollama run qwen3.5:4b`), and vLLM suits higher-throughput serving.
- **Faster decoding:** if your runtime supports it, try the Gemma 4 MTP drafters or Qwen3.6 MTP speculative decoding before buying more hardware.

## What to Watch Next

- **Mistral Large 4:** preview launched October 6, with open weights promised by the end of October. Reports put it at roughly 1T total and 49B to 52B active parameters, and the figures differ between sources.
- **Reflection Beam:** unveiled October 5, with Apache 2.0 weights promised this month.
- **Qwen4:** in training per Alibaba's September 22 announcement, with no date. Qwen3.8-Flash-Next previews its architecture.

These are large models, not edge models, but they matter to small-model users because strong open teachers tend to produce better distilled students.

## Takeaway

Small models in 2026 are no longer a compromise for simple tasks. Gemma 4's multimodal edge models and 12B Unified, the Qwen3.5 small series, and new coding-focused MoE models now cover agentic, multilingual and vision workloads locally. The best choice depends on your hardware ceiling, modality needs and license requirements, so benchmark two or three candidates on your own data before committing.

*This space changes quickly. Verify sizes, context lengths, dates and licenses against each vendor's model card before you build on them.*
