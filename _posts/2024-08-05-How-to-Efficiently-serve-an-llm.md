---
title: How to Efficiently Serve an LLM?
date: 2024-08-05 +05:30
categories: [LLM, Inference, Optimization, Serving]
tags: [LLM, inference, optimization, serving]
author: tremo
---

## How to Efficiently Serve an LLM

LLMs, or **Large** Language Models, are named so because they can range from tens to hundreds of billions of parameters. Their utility is clear, as LLMs are setting new records on various benchmarks and now often match or exceed human performance in multiple tasks [GPT-4 Technical Report](https://arxiv.org/html/2303.08774v4). Consequently, many companies are eager to deploy them in production. However, due to the unprecedented size of LLMs, there are significant challenges in serving them, such as slow token generation (tokens/second), memory limits for loading model parameters, KV cache (explained later), compute limits, and more. In this article, we will cover several recent ideas to help set up a robust LLM serving system.

![LLM Serving Overview](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/inference-engines-and-servers-architecture.png)__LLM Serving Overview__

## LLM inference Steps

1. Multiple users send requests to the LLM Server through HTTPs/gRPC.
2. The LLM Server receives the requests and schedules them based on QoE definition.
   - **QoE (Quality of Experience)**: Defined by two metrics:
     - **TTFT (Time to First Token)**: The time it takes for a user to receive the first response token.
     - **TDS (Token Delivery Speed)**: The rate at which the user receives tokens, which should be uniform and above the reader’s reading speed for a positive user experience.
    
    ![QoE](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/QoE%20Intro.png)__QoE Aware LLM Serving__
    
3. After scheduling, the LLM Inference process is divided into two phases:
   - **Prefill phase**: The LLM processes the input tokens in parallel and generates the output activations known as the “KV Cache”. This step is highly efficient at utilizing the GPU's parallel processing capabilities, making input tokens generally much cheaper than output tokens (as seen in the GPT-4o pricing chart). This phase produces the first output token and is typically compute-bound.
    
   - ![gpt-4o pricing](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/gpt-4o%20pricing.png)
   __GPT-4o Pricing__
    
   - **Decode phase**: The LLM starts autoregressively generating output tokens one at a time. This phase is slower in terms of inference and is where optimizations are **necessary**. Output tokens at each step are concatenated with the previous tokens’ KV cache to generate the next token.
    
   - ![KV Cache](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/KV%20Caching%20explanation%20&%20reuse.png)
   __KV Cache Explanation & Reuse__

## Optimizations

Many experts are innovating the inference stack, and multiple startups are competing to reduce costs to attract more customers.

- ![llama405-pricing](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/Llama%20405%20pricing%20by%20different%20providers.jpg)__LLAMA 405 Pricing by Different Providers__

Here are some interesting optimizations shared recently in research:

1. **Batching**:
   - Instead of serving one request at a time and wasting compute resources (since the decode phase has low arithmetic intensity and is memory-bound), we can amortize the cost of retrieving weights and KV cache from memory by serving multiple requests simultaneously.
   - ![Continuous Batching](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/continuous-batching.png)__Continuous Batching__

2. **Model Quantization (FP8/INT8)**:
   - Decreasing the precision of model weights and/or activations ([AWQ](https://arxiv.org/abs/2306.00978)/[GPTQ](https://arxiv.org/abs/2210.17323)) frees up more GPU VRAM, which allows for serving larger batches of requests.
   - ![Model Quantization](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/Quantaization.png)__Model Quantization__
3. **Paged Attention**:
   - The core idea behind [vLLM](https://docs.vllm.ai/en/latest/index.html), the most popular open-source serving engine, is to avoid memory fragmentation that occurs due to preserving the max context length for every request by using paging (borrowed from OS paging) to manage memory efficiently.
   - ![Paged Attention](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/PagedAttention%20VLLM.png)
   __Paged Attention in vLLM__
4. **Prefill Chunking / Stale-free Batching**:
   - Proposed by the [Sarathi-Serve paper](https://www.usenix.org/system/files/osdi24-agrawal.pdf), dividing the prefill context into smaller chunks allows merging the prefill and decode phases of different requests in the same batch.
   - ![Sarathi-Serve](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/Prefill%20decode%20prioritizing.png)
   __Prefill Decode Prioritizing__
5. **Prefill/Decode Disaggregation**:
   - In constrast to the previous idea, this paper [Mooncake: A KVCache-centric Disaggregated
Architecture for LLM Serving](https://arxiv.org/pdf/2407.00079) proposes separating the prefill and decode phases and transferring KVCache through a specialized design.
   - ![KVCache transfer in disaggregated arch](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/KVCache%20transfer%20in%20disaggregated%20arch.png)
   __KVCache Transfer in Disaggregated Architecture__
6. **KVCache Compression**:
   - As proposed by [CacheGen](https://arxiv.org/pdf/2310.07240), compressing the KVCache to speed up network transfer. This approach is beneficial for use cases with large context lengths (i.e., content summarization) which are over 16k input tokens to justify the encoding/decoding CPU overhead.
   - ![KVCache Compression](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/KV%20Compression.png)
   __KV Cache Compression__
7. **Speculative Decoding**:
   - Using extra smaller model(s) that generate tokens fast and in parallel. Selecting the output that matches the original model can speed up inference for simple use cases. **Note that** as the request batch size increases, the speed-up of speculative decoding diminishes.
   - ![Speculative Decoding](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/Speculative%20Decoding.png)__Speculative Decoding__
8. **Radix Attention (Prefix Caching)**:
   - This is the idea behind SGLang ([SGLang: Efficient Execution of
Structured Language Model Programs](https://arxiv.org/pdf/2312.07104)), which involves creating a data structure similar to a Prefix tree (Trie) for the KVCache to help reuse KVCache without recomputation. This only works for some use cases, like those shown in the image below:
   - ![Radix Attention](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/KV%20Cache%20sharing%20examples.png)__KV Cache Sharing Examples__
9. **Early Rejection**:
   - Predicting if a request can be served once received to avoid wasted resources (i.e., the server successfully computed the prefill part but failed at the decode phase due to memory limitations) will help improve server resource utilization and prevent downtime.
   - ![Early Rejection](/assets/img/posts/2024-08-05-How-to-Efficiently-serve-an-llm/Early%20Rejection%20based%20on%20prediction.png)__Early Rejection based on Prediction__

## Conclusion

Efficiently serving large language models is essential for businesses to reduce costs and increase generation speed (tokens/second). This efficiency opens the door for more use cases for LLMs. With the ideas presented here, you can optimize your LLM inference stack to achieve these goals and more!


## References

1. [Improving LLM Inference with Prefill Chunking / Stale-free batching (USENIX)](https://www.usenix.org/system/files/osdi24-agrawal.pdf)
2. [Mooncake: A KVCache-centric Disaggregated
Architecture for LLM Serving](https://arxiv.org/pdf/2407.00079)
3. [KVCache Compression and Streaming for Faster LLM Serving (arXiv)](https://arxiv.org/pdf/2310.07240)
4. [Dynamic Memory Management for LLMs: vAttention (arXiv)](https://arxiv.org/pdf/2405.04437)
5. [Enhancing Quality-of-Experience in LLM-Based Services (arXiv)](https://arxiv.org/pdf/2404.16283)
6. [Prefix Caching for Efficient LLM Inference (arXiv)](https://arxiv.org/pdf/2312.07104)
7. [Mastering LLM Techniques: Inference Optimization (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
8. [Token Probability Distribution (Hugging Face)](https://huggingface.co/spaces/osanseviero/token_probability_distribution)
9. [Welcome to vLLM! — vLLM Documentation](https://docs.vllm.ai/en/stable/)
10. [Serving Large Language Models: Technologies and Choices (run.ai)](https://www.run.ai/blog/serving-large-language-models)
11. [Efficient Large Language Model Serving (arXiv)](https://arxiv.org/pdf/2402.16363)
