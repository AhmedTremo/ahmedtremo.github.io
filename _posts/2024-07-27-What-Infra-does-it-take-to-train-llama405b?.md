---
title: What Infrastructure does it take to train a 405B Llama3-like model?
date: 2024-07-28
categories: [LLM, Infrastructure, GPU, Distributed Training]
tags: [LLM, infrastructure, GPU, distributed training]
author: tremo
---

## Intro

Setting up the infrastructure for training the latest frontier models is not an easy feat; only a few companies have the scale to do it (Meta, Microsoft, Google, ...). ML training has escalated from requiring up to [512 GPUs to needing 16k H100 GPUs](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/) to train Meta's latest Llama3-405B model. This posed a huge challenge for infrastructure setup, necessitating significant innovation to handle this sheer number of GPUs working in tandem, as LLM distributed training jobs require synchronous communication and gang scheduling.

Understanding the underlying infrastructure used to train the latest LLMs is essential for ML scientists to maximize the MFU (Model FLOPs Utilization), especially as infrastructure costs rise. AI labs are currently racing to build the first 100K H100 cluster that would cost an estimated [$4 billion](https://www.semianalysis.com/p/100000-h100-clusters-power-network). With that in mind, here’s an overview of the components required for building the infrastructure for the latest & greatest LLMs.

![Meta's 24k Cluster](/assets/img/posts/2024-07-27-What-Infra-does-it-take-to-train-llama405b/Infra%20Networking%20cluster.jpg)
__Meta’s 24k H100 cluster design with a 7:1 oversubscription ratio__

## Network Topology

The first and most important step is designing the networking flow of gradients across the huge number of GPUs. As aforementioned, distributed training requires synchronous communication methods like All-reduce, all-gather, and broadcast to combine and share gradients. As model sizes increase (reportedly [1.8 trillion](https://www.semianalysis.com/p/100000-h100-clusters-power-network) parameters for GPT-4), different parallelism techniques are required (Tensor, Context, Pipeline, Data) known as 4D parallelism that necessitate efficent design and communication.

In the ideal scenario, a GPU can communicate with any other GPU at full bandwidth (400 Gbps) using the latest Infiniband connection speed. However, achieving this for clusters of 100k GPUs would require a vast number of switches and transceivers to handle the communication traffic, making it cost prohibitive. To mitigate this, network architects trade-off by [oversubscribing](https://networkengineering.stackexchange.com/a/60003) the aggregation top layer (as shown in the figure of Meta’s 24K cluster design with a 7:1 ratio) to reduce the overall cost.

Deciding the communication patterns to be network-aware is essential to efficiently utilize the hardware and avoid stragglers (slower-performing nodes in a distributed system) that could slow down the entire cluster. For example, Meta forked Nvidia’s NCCL library to optimize the communication patterns to fit their cluster design.

## Storage

Training LLMs is memory-bound. While compute capabilities have rapidly evolved from different versions of GPUs (A100 → H100), the maximum memory capacity per GPU has increased, though not as dramatically as compute power. For example, A100 GPUs typically have up to 80 GB of HBM2e memory, whereas H100 GPUs can have up to 80 GB or more of HBM3 memory. More memory is essential for storing model weights, activations, and optimizer states (with Adam being the most popular optimizer, storing 3x parameters: one for the parameters themselves, one for the first moment (mean of gradients), and one for the second moment (variance of gradients)). With the rumored size of GPT-4 (1.8 trillion parameters), a total of 10.8 terabytes of memory would be required for training.

Additionally, memory is required for checkpointing (saving model weights frequently) to recover in case of failure or to choose the best-performing version of the model (if the model starts overfitting the data with more training). there are two ways to checkpoint:

- **Traditional way**: offloading to CPU memory and then to persistent storage (adds delay but is simpler to do).
- **Recent way**: using spare GPUs’ HBM to just RDMA copy the current model state for checkpointing; fast but require extra GPUs.

Storing datasets → 15.6 trillion tokens for Llama-3 required building 240 PB Storage for training, and fast data read speeds are needed to avoid wasting GPU cycles.

To ensure efficient training, the following aspects of storage must be considered:

1. **High I/O Throughput**: Ensuring fast data read and write speeds to avoid GPU downtime.
2. **Scalability and Fault Tolerance**: The storage system must scale with the growing dataset size and ensure data redundancy to protect against hardware failures.
3. **Integration with Compute**: Seamless integration with the compute infrastructure to allow high-speed data transfer between storage and GPUs.
4. **Intermediate and Result Storage**: Handling storage for intermediate results, logs, and multiple versions of model checkpoints efficiently.

## Compute

Nvidia is the leader in compute with an estimated market share of 80% to 90% and becoming the most valuable company in the world. H100s are currently in mass production and were used in training Llama3-405B, with most AI labs competing to build using H100s due to their leading performance and AI-friendly software stack (CUDA & cuDNN). AMD is trying to gain a share in this market with their MI250, and cloud providers are starting to build their own chips. Google’s TPU chips, used to train the Gemini family of models, are notable, but it doesn’t seem the market will change significantly in the foreseeable future.

| Feature             | Nvidia H100 | AMD MI250  | Google TPU v4 |
|---------------------|-------------|------------|---------------|
| Architecture        | Hopper      | CDNA 2     | TPU           |
| Memory (HBM)        | 80 GB       | 128 GB     | 128 GB        |
| Memory Bandwidth    | 3.0 TB/s    | 3.2 TB/s   | 2.7 TB/s      |
| Peak FP16 Throughput| 198 TFLOPS  | 383 TFLOPS | 275 TFLOPS    |
| NVLink              | 900 GB/s    | -          | -             |


## Fault Tolerance & Health Checks

Ensuring fault tolerance and performing regular health checks are crucial for maintaining the stability and efficiency of a large-scale AI training infrastructure. Given the high failure rates of GPUs and other components, robust fault tolerance mechanisms are essential to maximize hardware utilization and minimize downtime.

### Key Fault Tolerance Strategies

1. **Spare Capacity**:
   - [Imbue Recommendation](https://imbue.com/research/70b-infrastructure/) is to maintain 10-20% more spare GPUs than required which allows quick replacement of failed GPUs, ensuring the training run is not halted due to a single failure.

2. **Health Checks**:
   - Implement scripts to detect faulty hardware (GPUs, InfiniBand, host machines, etc...).

3. **Network Reliability**:
   - Networks can fail due to flapping, host machine failures, or power supply issues, you need to use redundant paths and automatic failover mechanisms to ensure continuous operation.

4. **Golden Sets of Machines**:
   - Maintain a set of machines that have been stress-tested and verified to be reliable by running stress tests that maximize hardware utilization to distinguish between great and med machines.

5. **Checkpointing**:
   - Use spare GPUs' HBM to RDMA (Remote Direct Memory Access) copy the current model state, offering a faster but more costly alternative. This method leverages the high bandwidth memory (HBM) of GPUs and the fast RDMA transfer speeds, reducing the time required for checkpointing.

6. **Proactive Checks & Automated Recovery**:
   - Regularly measure and log power consumption, temperature, and fan speed.
   - Develop scripts for automated recovery processes, such as switching to spare GPUs and rerouting network traffic.


To illustrate the common issues faced during large-scale training runs, here are the interruption categories Meta encountered during their 54-day training run for Llama3:

![Meta Interruptions](/assets/img/posts/2024-07-27-What-Infra-does-it-take-to-train-llama405b/Meta%20interruptions.png)
__Meta’s interruptions during 54-day training run__

## Conclusion

Understanding the infrastructure needs for training the frontier models might seem like a niche skill that only a few engineers in AI labs need. This is only true until an ML scientist faces a hardware error, which is inevitable due to the high percentage of failures of current hardware. Knowledge of the underlying infrastructure can help navigate these challenges with ease. Moreover, building software for training that fully utilizes the strengths and avoids the weaknesses in the infrastructure cluster will save a lot of money and might just be the differentiator between you and your competitors in this fast-paced space.

## References

1. [100k H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing (semianalysis.com)](https://www.semianalysis.com/p/100000-h100-clusters-power-network)
2. [The LLaMA 3 Herd of Models (Meta AI)](https://ai.meta.com/research/publications/the-llama-3-herd-of-models/)
3. [Building Meta's GenAI Infrastructure (Meta Engineering)](https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/)
4. [Training Large Language Models at Scale (Meta Engineering)](https://engineering.fb.com/2024/06/12/data-infrastructure/training-large-language-models-at-scale-meta/)
5. [70B Infrastructure (Imbue)](https://imbue.com/research/70b-infrastructure/)
