# Prefill–Decode Disaggregation

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

**Prefill–decode disaggregation** places prompt processing and token generation on separate worker pools. It is a system architecture choice intended to reduce interference and let each phase scale independently—not a change to the model's prefill/decode semantics.

First understand the phases and their metrics in [Generation lifecycle and performance metrics](generation-lifecycle-and-metrics.md), then the state that must move in [KV cache and memory management](kv-cache-and-memory.md).

## Why separate the phases?

Prefill processes the known prompt and creates its KV state; it often performs large parallel matrix operations. Decode advances one token at a time and repeatedly reads the request's KV state along with model weights. Under some workloads, a large prefill on a shared device can delay the next decode step for other users, while a heavily occupied decode pool can make a new prompt wait.

The symptoms map roughly to:

- **TTFT:** includes request queueing and prompt-side work, so it can rise when prefill capacity is insufficient.
- **Inter-token latency / TPOT:** can rise when decode is delayed by scheduling or competing prefill work.

The terms "compute-heavy prefill" and "memory-heavy decode" are useful workload models, not immutable properties of all deployments. Batch size, model, sequence length, kernels, and hardware can change the bottleneck.

## Request flow

```text
client request
      |
      v
router / admission scheduler
      |
      +----> prefill worker pool
      |       process prompt; produce first-token distribution/token
      |       construct prompt KV state
      |                 |
      |                 | transfer compatible KV state over interconnect
      |                 v
      +----> decode worker pool
              load/attach KV state; generate subsequent tokens
                         |
                         v
                 stream response to client
```

Real systems may pipeline KV transfer with prefill or decode and may transfer blocks incrementally. The diagram shows the logical stages, not a promise that each stage waits for the previous one to finish completely.

## The transfer is part of the serving path

KV state can be large. A rough lower bound for transfer time is:

```text
transfer time ≈ KV bytes / effective link bandwidth
```

In practice, setup, queueing, synchronization, contention, serialization/layout conversion, and concurrent transfers matter too. Disaggregation helps only when the reduction in interference and improved resource utilization outweigh the cost of coordination and moving cache state. The prefill and decode workers must agree on compatible model weights, cache representation, position/state semantics, and request metadata.

Operationally, the router must coordinate capacity in both pools, account for cache-transfer time and failure, and avoid sending work to a decode pool that cannot retain its KV state. Pool imbalance can leave one side saturated while the other is underused.

## What it can improve—and what it costs

| Potential benefit | Corresponding cost or condition |
| --- | --- |
| Scale prefill and decode capacity independently for prompt-heavy or generation-heavy traffic | Requires enough sustained traffic and reliable phase-specific capacity planning |
| Reduce prefill/decode interference on shared accelerators | Requires routing, admission control, transfer, and more complex operations |
| Select different hardware or parallelism strategies per phase | Model and cache compatibility constrain which configurations can hand off state |
| Improve predictability for decode under heavy prefill load | KV movement can add latency and consume interconnect bandwidth |
| Tune each pool to its own request mix | Skewed traffic or inaccurate routing can create an idle pool and a queue on the other |

### When to consider it

Consider disaggregation when measurements show sustained contention between large prefills and active decode, phase-specific scaling would materially improve utilization, and the interconnect can move KV state within the latency budget. Compare it against simpler options such as continuous batching, chunked prefill, prefix reuse, and adding co-located capacity.

It is often unnecessary for small deployments, low or bursty traffic, short contexts, or cases where KV transfer dominates. A simple co-located server can be cheaper and easier to operate.

## How to evaluate

Run the same representative workload against co-located and disaggregated configurations. Record TTFT, inter-token latency percentiles, end-to-end latency, input/output throughput, queue depth in both pools, KV bytes and transfer latency, link utilization, accelerator utilization, and total cost. Include burst traffic and failures in addition to steady-state runs. Do not compare only peak throughput or one averaged latency number.

## References

- Amit Shekhar, [Prefill-Decode Disaggregation](https://outcomeschool.com/blog/prefill-decode-disaggregation), [Prefill vs Decode: LLM Inference Optimization](https://outcomeschool.com/blog/prefill-vs-decode-llm-inference-optimization), and [How does SGLang work?](https://outcomeschool.com/blog/how-does-sglang-work).
- [LLM Inference Engineering repository](https://github.com/amitshekhariitbhu/llm-inference-engineering), Parts 1 and 3.
- JAX Scaling Book, [Inference](https://jax-ml.github.io/scaling-book/inference/) chapter, for analytical trade-offs, KV sharding, and serving-engine designs. Its hardware examples are specific to the systems modeled there.
- Google, [JetStream repository](https://github.com/AI-Hypercomputer/JetStream), as an implementation case study. Its README carries an archival/migration notice; follow the linked [vLLM JAX TPU project](https://github.com/vllm-project/tpu-inference) for current project status rather than treating JetStream as an actively maintained default.
- SGLang, [official documentation](https://docs.sglang.io/); serving capabilities and configuration are version-dependent.
- vLLM, [official documentation](https://docs.vllm.ai/en/stable/); check release-specific disaggregated-serving support and requirements.
- NVIDIA, [Dynamo documentation](https://docs.nvidia.com/dynamo/latest/) for one example of a distributed-serving orchestration system; check the current version for supported engine integrations.

Sources accessed 2026-10-30.
