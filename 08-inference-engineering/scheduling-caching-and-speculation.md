# Scheduling, Caching, and Speculative Decoding

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

An inference server must decide which requests share each model execution step, how much prompt work to admit, how to reuse repeated work, and how to deliver generated tokens. These are runtime and scheduling choices around the model's autoregressive loop; they are not all the same kind of optimization.

## From static batches to continuous batching

With **static batching**, a group of requests is formed and run together. Because requests have different prompt and output lengths, short requests may finish while the rest keep the batch occupied. The system may also reserve resources for a long sequence up front.

**Continuous batching** (often called iteration-level scheduling) revisits the active set at generation steps. When a request finishes or releases capacity, the scheduler can admit a waiting request instead of waiting for the whole original batch to complete. It improves device utilization for variable-length requests, but scheduler decisions still affect fairness, queueing, and tail latency.

```text
Static batch:       [A, B, C] ---------------- wait for the slowest request
Continuous batch:   [A, B, C] -> [A, C, D] -> [A, D, E]
                              B finished; D joins   C finished; E joins
```

The diagram is conceptual: a real scheduler also considers prefill tokens, KV-block availability, priorities, and fairness. Continuous batching removes the need to keep a completed request occupying a decode slot, but it does not remove queueing or guarantee a particular latency.

Continuous batching relies on per-request state, including each sequence's KV cache. Paged KV allocation helps a runtime manage that state efficiently; see [KV cache and memory management](kv-cache-and-memory.md#pagedattention-allocate-kv-state-as-blocks).

### Prefill can disrupt decode

A long prompt requires a comparatively large prefill operation. If that work runs on the same device as active decode iterations, it may delay their next token. **Chunked prefill** divides prompt work into smaller chunks so a scheduler can interleave prompt tokens with decode work. This can reduce long pauses, but sharing the device still creates a trade-off: smaller chunks may prolong prompt completion, while larger chunks can again interrupt decode. Exact policy and effect depend on the workload and runtime.

For persistent interference at production scale, compare chunked scheduling with [prefill–decode disaggregation](prefill-decode-disaggregation.md).

## Prefix caching

When requests have a shared, exactly matching prefix—such as the same system prompt—the runtime may reuse the prefix's computed KV state. This avoids repeating some prefill work and can lower TTFT. It is useful only when the prefix and execution context are compatible; semantically similar but differently tokenized prompts are not automatically interchangeable.

The term **prompt caching** is used inconsistently. In this note it means reuse of model-side prefix computation/KV state, not generic response caching. See [KV cache and memory management](kv-cache-and-memory.md#prefix-sharing-avoid-duplicate-prefill-state) for the conditions and memory implications.

## Speculative decoding: draft, then verify

The target model is still responsible for the final output, but a cheaper process proposes several future tokens:

1. A draft model, auxiliary prediction heads, or a token-matching method proposes candidates.
2. The target model evaluates the candidates together using a parallel verification pass.
3. A compatible verification/sampling procedure accepts a prefix of candidates and corrects at the first rejection.

Under the standard speculative-sampling construction, the output distribution can be preserved: for draft distribution `q` and target distribution `p`, a proposed token can be accepted with probability `min(1, p(x)/q(x))`; on rejection, the next token is sampled from the normalized residual `max(0, p - q)`. Exact algorithms and implementation details matter—an approximate multi-token method does not automatically preserve the target distribution.

The main variants in the source collection differ in how they create proposals:

- **Draft-model speculation:** a smaller model autoregressively generates a short candidate continuation. The target evaluates the prompt plus candidate tokens in parallel, accepting the verified prefix and correcting at a rejection.
- **N-gram speculation:** match a recent token suffix against earlier text in the prompt/context or output, then propose the continuation that followed the match. It avoids loading a separate learned draft model, but helps most when the workload repeats patterns.
- **Medusa:** auxiliary heads attached to the target propose tokens at different future positions. Candidate branches form a tree; tree-aware attention lets the target verify compatible paths in one pass. More candidate heads/branches increase proposal coverage but also consume compute and memory.
- **EAGLE:** a lightweight draft component predicts future model features rather than guessing only token IDs, then uses those features to build candidate branches for target verification. It still depends on good acceptance and the verifier; it is not a separate full-size replacement model.

The potential gain depends on how many proposed tokens are accepted, the cost of proposing and verifying, available parallelism, and memory use. Measure accepted tokens per target pass and net latency on your own prompt/output distribution. Treat model-specific speedup claims in source material as illustrative, not as guaranteed results.

## Prefix reuse structures

The serving system needs both (1) a way to allocate K/V memory and (2) a way to find which cached prefix a new request can reuse. Paged KV blocks solve allocation; a prefix index solves lookup. One design is a radix tree/trie whose paths represent token prefixes and whose nodes point to reusable KV blocks. The runtime finds the longest compatible cached prefix, attaches those blocks to the request, and prefills only the unmatched suffix.

The cache needs eviction when capacity is scarce, and a cache hit must include all compatibility conditions described in [KV cache and memory management](kv-cache-and-memory.md#prefix-sharing-avoid-duplicate-prefill-state). Prefix lookup does not make non-identical prompts equivalent. This distinction helps explain why vLLM's paged block management and SGLang's RadixAttention are related but not the same mechanism.

## Streaming is delivery, not faster inference

With **token streaming**, the service sends each output token (or chunk) as it becomes available rather than waiting for the complete response. This lowers perceived time to first content and exposes the model's generation cadence, but does not by itself reduce prefill time or decode computation. Network buffering, detokenization, and frontend rendering can change the observed inter-token intervals.

## Choose by objective

| Technique | Primary opportunity | Important cost or risk |
| --- | --- | --- |
| Continuous batching | Higher utilization across variable-length requests | Scheduling/fairness trade-offs and KV-state pressure |
| Chunked prefill | Reduce long-prefill interruption to decode | May lengthen prefill completion or add scheduling overhead |
| Prefix caching | Avoid repeated work for identical compatible prefixes | Consumes cache memory; limited by prefix reuse and compatibility |
| Speculative decoding | More accepted output tokens per target-model iteration | Draft/verification overhead, memory cost, and acceptance-rate dependence |
| Token streaming | Earlier visible response and incremental UI updates | Does not make token computation faster; transport can add buffering |

## References

- Amit Shekhar, [Continuous Batching in LLMs](https://outcomeschool.com/blog/continuous-batching-in-llms), [How does Prompt Caching work?](https://outcomeschool.com/blog/how-does-prompt-caching-work), [How does Token Streaming work?](https://outcomeschool.com/blog/how-does-token-streaming-work), and [Speculative Decoding](https://outcomeschool.com/blog/speculative-decoding).
- Amit Shekhar, [Decoding Medusa](https://outcomeschool.com/blog/decoding-medusa), [Decoding EAGLE](https://outcomeschool.com/blog/decoding-eagle), and [N-gram Speculation in LLMs](https://outcomeschool.com/blog/n-gram-speculation-in-llms).
- Yaniv Leviathan et al., [Fast Inference from Transformers via Speculative Decoding](https://arxiv.org/abs/2211.17192).
- Tianle Cai et al., [Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads](https://arxiv.org/abs/2401.10774); Yuhui Li et al., [EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty](https://arxiv.org/abs/2401.15077).
- vLLM, [feature documentation](https://docs.vllm.ai/en/stable/); exact support and configuration depend on release, model, and hardware.

Sources accessed 2026-10-30.
