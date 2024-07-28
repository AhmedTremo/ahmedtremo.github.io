---
title: What Infrastructure does it take to train a 405B Llama3-like model?
date: 2024-07-28
categories: [LLM, Infrastructure, GPU, Distributed Training]
tags: [LLM, infrastructure, GPU, distributed training]
author: tremo
---

## Intro

Setting up the infrastructure for training the latest frontier models is not an easy feat; only a few companies have the scale to do it (Meta, Microsoft, Google). ML training has escalated from requiring up to 512 GPUs to needing 16k H100 GPUs to train Meta's latest Llama3-405B model. This posed a huge challenge for infrastructure setup, necessitating significant innovation to handle this sheer number of GPUs working in tandem, as LLM distributed training jobs require synchronous communication, known as gang scheduling.

Understanding the underlying infrastructure used to train the latest LLMs is essential for ML scientists to fully utilize it, especially as infrastructure costs rise. For example, AI labs are racing to build a 100K H100 cluster that would cost an estimated $4 billion (source: [100k H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing (semianalysis.com)](https://www.semianalysis.com/p/100000-h100-clusters-power-network)). With that in mind, here’s an overview of the components required for building the infrastructure for the latest LLMs.

![Meta's 24k Cluster](/assets/img/posts/2024-07-27-What-Infra-does-it-take-to-train-llama405b/Infra%20Networking%20cluster.jpg)
__Meta’s 24k cluster design with a 7:1 oversubscription ratio__

## Network Topology

1. **Network Topology:**
    1. The first and most important step is designing the networking flow of data across the huge number of GPUs. As aforementioned, distributed training requires synchronous communication methods like All-reduce, all-gather, and broadcast to combine and share gradients. As model sizes increase (reportedly 1.7 trillion parameters for GPT-4), different parallelism techniques are required (Tensor, Context, Pipeline, Data) that necessitate more communication.
    2. In the ideal scenario, a GPU can communicate with any other GPU at full bandwidth (400 GB/s) for the latest NVLink connection speed, known as full-bisection connection. However, doing so for clusters of 100k GPUs would require a huge number of switches and transceivers to route the communication traffic, making the cost prohibitive. Architects thus trade-off by oversubscribing the aggregation top layer (as shown in the figure of Meta’s 24K cluster design with a 7:1 ratio) to decrease the cluster cost.
    3. GPUs within the same rack have full bisection bandwidth with one another. Therefore, deciding the communication patterns to be network-aware is essential to efficiently utilize the hardware and avoid stragglers that could slow down the entire cluster. For example, Meta forked Nvidia’s NCCL library to optimize the communication patterns to fit their cluster design.

## Storage

2. **Storage:**
    1. Training LLMs is memory-bound. While compute has rapidly evolved from different versions of GPUs (A100 → H100), memory has almost stayed the same (80 GB max) on both chips. More memory is essential for storing model weights, activations, and optimizer states (with Adam being the most popular optimizer, storing 3x parameters). With the rumored size of GPT-4 (1.7 trillion parameters), a total of X terabytes would be required.
    2. Memory is within the chip.
    3. Additionally, memory is required for checkpointing (saving model weights frequently) to recover in case of failure or to choose the best-performing version of the model (if the model starts overfitting the data with more training).
        1. Traditional way: offloading to CPU memory and then to persistent storage (adds delay but is easier to do).
        2. Recent way: using spare GPUs' HBM to copy the current model state for checkpointing; fast but costly.
    4. Storing datasets → 15 trillion tokens for LLaMA required (X Storage) for training, and fast data read speeds are needed to avoid wasting GPU cycles.

## Compute

3. **Compute:**
    1. Nvidia is the leader in compute (market share and company value). H100s are currently in mass production and were used in training Llama405B, with most AI labs competing to build using H100s due to their leading performance and AI-friendly software stack (Cuda & NCCL). AMD is trying to gain a share in this market with their MI200, and cloud providers are starting to build their own chips. Google’s TPU chips, used to train the Gemini family of models, are notable, but it doesn’t seem the market will change significantly in the foreseeable future.

## Fault Tolerance & Health Checks

4. **Fault Tolerance & Health Checks:**
    1. GPUs fail, and they fail a lot. According to Imbue, 10-20% of GPUs fail, and this isn’t the only source of failure in a large AI cluster. Networks can fail (flapping), host machines can die, power supplies may fluctuate, and even the current temperature can affect cluster throughput (source: Meta). Designing infrastructure to be fault-tolerant is essential to maximize hardware utilization. Imbue recommends having 10-20% more spare GPUs than required for training to quickly discard failed GPUs and use spare ones, to avoid stopping the full cluster due to a single GPU or cable failure halting the synchronous training run. Below is a categorized list of interruptions Meta’s team faced during their 54-day training run.
    2. Health checks: creating scripts to check every single component is essential to automatically detect faulty hardware (GPU, InfiniBand, host machine, etc.). Automatic detection should be followed by automatically excluding faulty hardware and using another spare one. Tickets should be sent to data center technicians or vendors for fixes, and only when confirmed fixed, should the hardware be re-added to the training cluster.
    3. Building a list of golden sets of machines and networks is important to narrow down failure sources. Running stress tests that maximize hardware utilization will reveal great from good machines.

![Meta Interruptions](/assets/img/posts/2024-07-27-What-Infra-does-it-take-to-train-llama405b/Meta%20interruptions.png)
__Meta’s interruptions during 54-day training run__

## Conclusion

Understanding the infrastructure needs for training the frontier models might seem like a niche skill that only a few engineers in AI labs need. This is only true until an ML scientist faces a hardware error, which is inevitable due to the high percentage of failures of current hardware. Knowledge of the underlying infrastructure can help navigate these challenges with ease. Moreover, building software for training that fully utilizes the strengths and avoids the weaknesses in the infrastructure cluster will save a lot of money and might just be the differentiator between you and your competitors in the space.
