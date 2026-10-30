# Generation Lifecycle and Performance Metrics

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

An autoregressive language model produces a probability distribution for the next token, selects a token, appends it to the sequence, and repeats. That dependency makes generation partly sequential, even though substantial computation within each model step is parallel. Inference systems divide a request into **prefill** and **decode** because those phases have different work shapes and often stress different resources.

## From text to generated tokens

The request first becomes token IDs using the model's tokenizer. The model consumes those IDs along with its parameters and produces logits (scores over the vocabulary). A decoding policy turns scores into the next token: greedy selection, sampling with temperature/top-p, beam search, or another supported method. The selected token is appended to the context and fed back to generate the next one.

This is **autoregressive generation**: the next token depends on the preceding sequence. It explains both why output is generated step by step and why techniques such as speculative decoding try to verify multiple candidate future tokens together.

## The two phases

### Prefill

Prefill processes the available input sequence: the system prompt, conversation history, user message, and any supplied context. The input sequence is known, so a model can process its positions in parallel within the causal-attention constraints. It computes the prompt's layer-by-layer attention state and writes the corresponding keys and values into the KV cache. The resulting logits provide the distribution for the first generated token.

Longer prompts generally require more work and create more KV state. Attention over a full sequence has pairwise interactions, so the attention component has quadratic work in sequence length for standard dense attention; optimized kernels reduce data movement, not the underlying exact attention relationships. Other model operations and implementations also contribute to prefill cost.

### Decode

After the first token, each decode iteration consumes the latest token, attends to the request's prior keys and values, and produces the next-token distribution. The new key and value are appended to the cache. Because each generated token depends on the preceding output, a single sequence advances sequentially.

For a small decode batch, each step has little token-parallel work relative to the model weights and growing KV state it must access. Decode is therefore often sensitive to memory bandwidth and cache traffic. Larger batches can reuse weight reads across sequences and alter the bottleneck, so "decode is memory-bound" is a useful starting hypothesis, not a law for every model, batch, or accelerator.

## A request timeline

```text
request arrives
      |
      v
queueing / tokenization / input preparation
      |
      v
prefill(prompt) ---> prompt KV state + logits ---> sample first output token
      |                                           |
      +-------------------- TTFT -----------------+
                                                  |
                         decode one token at a time
                         read old KV, append new KV
                                                  |
              token stream: t1, t2, t3, ... ------+
                                                  v
                                       stop condition / response done
```

The KV cache is the bridge between phases; see [KV cache and memory management](kv-cache-and-memory.md) for its contents and memory footprint.

## Metrics: measure the experience and the system

| Metric | What it describes | Interpretation |
| --- | --- | --- |
| **TTFT — time to first token** | Time from request arrival (or another explicitly defined start point) to the first generated token being available | Captures queueing and prompt-side work as well as generation setup; do not assume it is only prefill compute. |
| **Inter-token latency (ITL)** | Time between successive output tokens for one streamed response | A distribution or percentile view shows pauses that an average can hide. |
| **TPOT — time per output token** | A per-request or aggregate average of output-token intervals, depending on the benchmark | Check the benchmark's formula and whether it excludes TTFT; implementations use the term differently. |
| **End-to-end latency** | Time from a defined request start until the response completes | Depends on prompt processing, output length, scheduling, decoding, and transport. |
| **Throughput** | Work completed per unit time, such as input tokens/s, output tokens/s, or requests/s | Always state whether it is aggregate or per-request and include the load and latency target. |
| **Goodput / SLO attainment** | Requests completed within the required latency and quality limits | Useful when raw throughput would count responses that are too slow to be useful. |

For a single request with `N > 1` output tokens, some benchmarks calculate average TPOT as `(completion time - TTFT) / (N - 1)`. Other tools use different conventions, especially around the first token and streaming timestamps. Report the exact definition, token-count policy, start/end points, and percentiles.

### Read metrics together

- Low TTFT with high ITL means the answer starts quickly but then stalls or streams slowly.
- High TTFT and stable ITL points toward queueing, prompt processing, or long-prefill interference.
- High aggregate output-token throughput does not guarantee low per-user latency; large batches may improve device utilization while increasing queueing or tail latency.
- Compare p50 and tail percentiles under the same arrival pattern, prompt/output distribution, concurrency, model, and decoding policy.

## A bottleneck-oriented diagnosis

Before selecting an optimization, separate the stages: request queue, tokenization and input assembly, prefill, decode, KV allocation/transfer, output post-processing, and network streaming. Profile the relevant stage and resource. Examples:

- A prompt-heavy workload with poor TTFT suggests examining prefill throughput, queueing, chunking, and reusable prefixes.
- A long-output workload with high ITL suggests examining decode throughput, batch composition, KV bandwidth, and speculative decoding acceptance.
- Memory exhaustion at rising concurrency suggests estimating model weights and per-sequence KV state separately.
- A GPU with low compute utilization may still be busy moving data or waiting for the host, scheduler, or interconnect.

For memory estimation, continue to [KV cache and memory management](kv-cache-and-memory.md). For scheduling and decoding levers, see [Scheduling, caching, and speculative decoding](scheduling-caching-and-speculation.md).

## A simple compute-versus-memory model

A roofline estimate helps decide whether to investigate arithmetic throughput or data movement first. For an operation with `F` floating-point operations and `Q` bytes transferred, an idealized lower bound is:

```text
math time       ≈ F / peak FLOPs per second
memory time     ≈ Q / effective memory bandwidth
step time       ≥ max(math time, memory time)
arithmetic intensity = F / Q
```

The hardware's **ridge point** is approximately `peak FLOPs/s ÷ memory bandwidth`. Below that arithmetic intensity, data movement is the limiting term in the idealized model; above it, arithmetic is. Real execution also includes kernels, synchronization, communication, and imperfect overlap. Use sustainable measured rates when available, not advertised peak specifications.

For a rough decode-step estimate, let `B` be the number of active sequences, `P` the dense model's parameter count, `W` the model-weight bytes read per step, and `K` the KV bytes read for one sequence in that step. If a batch shares each weight read, a simplified lower bound is:

```text
decode step time ≳ max(
    2 × B × P / peak FLOPs per second,
    (W + B × K) / effective memory bandwidth
)
```

This leaves out attention arithmetic, activations, dequantization, communication, scheduling, and launch costs; it is a way to reason about competing terms, not an end-to-end latency prediction. `K` grows with the cached context, while the KV allocation across active requests grows roughly with `B × K`. In this model, the approximate batch-size crossover at which dense-model arithmetic catches up with weight reads is:

```text
B_critical ≈ W × peak FLOPs per second
             / (2 × P × effective memory bandwidth)
```

If weights occupy about `s` bytes per parameter, then `W ≈ P × s`; the crossover depends on precision and the hardware's compute-to-bandwidth ratio. KV traffic, kernel efficiency, and interconnect costs shift the real crossover. A batch that is compute-efficient may also exceed the memory budget or latency target.

Attention has a different shape. For one head, let `T` be the number of query positions processed together and `S` the number of key/value positions read. With an IO-aware kernel that does not materialize the full attention matrix, a simplified arithmetic-intensity estimate is:

```text
attention arithmetic intensity ≈ S × T / (S + T)
```

This estimate counts the Q/K/V and output traffic, and omits softmax/mask work and other implementation details. During prefill, `S = T`, so intensity grows as roughly `T / 2`; during single-token decode, `T = 1` and a long context gives intensity near one. That is why attention's KV reads often make decode bandwidth-sensitive even when the MLP's weight reuse has moved it toward the compute roofline. The expression is an explanatory model, not a guarantee that every kernel or accelerator has the same bottleneck; GQA/MQA, cache layout, sharding, and kernel behavior change the actual traffic.

This explains the common, but not universal, pattern: prefill can expose large, efficient matrix operations, while small-batch decode repeatedly reads weights and a growing KV cache for only a few new tokens. Larger decode batches amortize weight reads across sequences until arithmetic, KV bandwidth, capacity, or the service-level objective becomes limiting. For the cache's size and distributed layout, see [KV cache and memory management](kv-cache-and-memory.md).

## Benchmarking as a controlled experiment

Compare configurations under the same model/checkpoint, tokenizer, runtime version, precision, decoding settings, hardware, and request mix. The workload should reflect prompt- and output-length distributions, concurrency, prefix reuse, and arrival behavior; uniform synthetic requests can hide burst and queueing effects. Warm up before measurement, run multiple configurations, and report load conditions together with p50/p90/p99 TTFT, inter-token latency, completion latency, throughput, and SLO goodput.

Separate **serving benchmarks** from **profilers**:

- Serving/load tools measure the service path. Examples include NVIDIA [AIPerf](https://github.com/ai-dynamo/aiperf), vLLM's [`vllm bench serve`](https://docs.vllm.ai/en/stable/cli/bench/serve.html), and SGLang's [`bench_serving`](https://docs.sglang.io/docs/developer_guide/benchmark_and_profiling.html).
- Profilers help explain a measured result. [PyTorch Profiler](https://docs.pytorch.org/tutorials/recipes/recipes/profiler_recipe.html) can show operator time and memory; [Nsight Systems](https://developer.nvidia.com/nsight-systems) provides a system-wide timeline; [Nsight Compute](https://developer.nvidia.com/nsight-compute) provides detailed CUDA-kernel metrics.

The software-stack source also mentions GenAI-Perf; its [upstream repository](https://github.com/triton-inference-server/perf_analyzer/tree/main/genai-perf) now says it is being phased out and recommends AIPerf for new benchmarking needs. Check each tool's current documentation and version before comparing results.

## References

- [LLM Inference Engineering repository](https://github.com/amitshekhariitbhu/llm-inference-engineering), Part 1.
- Amit Shekhar, [Autoregressive Models](https://outcomeschool.com/blog/autoregressive-models).
- Amit Shekhar, [Prefill vs Decode: LLM Inference Optimization](https://outcomeschool.com/blog/prefill-vs-decode-llm-inference-optimization).
- Amit Shekhar, [How does Token Streaming work?](https://outcomeschool.com/blog/how-does-token-streaming-work).
- Amit Shekhar, [Tokenization in LLMs](https://outcomeschool.com/blog/tokenization-in-llms).
- Amit Shekhar, [Self Attention in Transformers](https://outcomeschool.com/blog/self-attention-in-transformers) and [Math behind Attention: Q, K, and V](https://outcomeschool.com/blog/math-behind-attention-qkv).
- JAX Scaling Book, [Inference](https://jax-ml.github.io/scaling-book/inference/) and [Roofline](https://jax-ml.github.io/scaling-book/roofline/) chapters. Their analytical models are useful for reasoning; the example hardware and benchmark values are not universal.
- JAX Scaling Book, [Applied Inference](https://jax-ml.github.io/scaling-book/applied-inference/) for a worked serving example.
- Aakanksha Chowdhery et al., [Efficiently Scaling Transformer Inference](https://arxiv.org/abs/2211.05102), an analytical study of large-model inference on TPU v4. Its measurements and hardware setup are historical examples, not current accelerator comparisons.
- Inference Engineering, [Software](https://inferenceengineering.tech/chapters/software/) chapter.
- NVIDIA, [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/).

Sources accessed 2026-10-30.
