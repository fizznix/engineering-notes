# KV Cache and Memory Management

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

The **key-value (KV) cache** stores attention keys and values already computed for a request. Without it, each new autoregressive step would need to recompute the keys and values for all earlier tokens. With it, each layer computes K and V for the new token and reuses the prior state. The cache reduces repeated computation, but trades it for memory that grows with context length and concurrent requests.

For the attention equation and how model architecture changes KV size, see [Attention kernels and model architectures](attention-kernels-and-model-architectures.md). For how schedulers allocate requests that need KV memory, see [Scheduling, caching, and speculative decoding](scheduling-caching-and-speculation.md).

## How it grows

In a standard decoder-only Transformer, every layer retains keys and values for the tokens still in the attention context. A useful first-order estimate for one request is:

```text
KV bytes ≈ 2 × layers × tokens × KV heads × head dimension × bytes per element
```

The factor of two is for K and V. For a batch, multiply by the number of active sequences, then account for implementation details such as block rounding, alignment, replicated state, sliding-window layers, multimodal state, and allocator metadata.

**Example estimate:** suppose a model has 32 layers, 8 KV heads, head dimension 128, and stores each KV element in 2 bytes. For 8,192 cached tokens:

```text
2 × 32 × 8,192 × 8 × 128 × 2 = 1,073,741,824 bytes ≈ 1 GiB
```

This is a deliberately simplified estimate for one sequence, not a measurement of a specific model. A batch of 16 such sequences would require about 16 GiB just for these KV values under the same assumptions, before runtime overhead or other memory use.

The cache size is affected by the **KV-head count**, not the query-head count. This is why grouped-query attention can reduce cache use; it is an architectural choice, not a runtime allocator trick.

## Shard KV state across devices

When one accelerator cannot hold or serve the KV state efficiently, a runtime can distribute the cache over a device mesh. This changes where the state lives and which devices perform attention; it does not eliminate the aggregate KV values. The layout must match the model's attention heads, active batch, context length, and the engine's parallelism plan.

Common dimensions to consider are:

- **KV-head sharding:** assign different KV heads to different devices. This fits naturally with model parallelism when there are enough KV heads. With MQA or GQA, the number of KV heads can be too small to divide across all devices.
- **Batch sharding:** place different requests' caches on different devices. This can use more devices when KV-head sharding runs out of heads, but the query activations and attention outputs may need collectives to move between the model- and batch-sharded layouts.
- **Sequence/context sharding:** split each request's cached positions across devices. This can help with long contexts or small batches, but attention has to combine partial results across the sequence shards, adding communication to the decode path.

These are layout choices, not free capacity multipliers: sharding can lower per-device memory and distribute KV reads, while collectives, layout conversion, and interconnect bandwidth add latency. Replication, uneven layouts, and limits on shardable heads can also prevent per-device memory from scaling linearly with device count. See [parallel inference across accelerators](hardware-and-kernels.md#scale-one-model-across-accelerators) for the broader communication trade-offs.

## PagedAttention: allocate KV state as blocks

A naive allocator can reserve one contiguous KV region sized for a request's maximum output length. The request may finish much earlier, leaving most of its reservation unused. Variable-length reservations can also leave free gaps too small to fit another large contiguous region. These are forms of internal and external fragmentation.

Paged KV management divides the sequence into fixed-size **blocks** (also called pages) and allocates physical blocks as tokens arrive. A per-sequence **block table** maps a logical block number to a physical block in the KV pool. Logical token order stays contiguous to the model even though physical memory blocks can be scattered.

```text
Example with four token slots per block:

Logical sequence A:  A B C D | E F
Block table A:       0 -> physical block 7
                     1 -> physical block 2

Logical sequence B shares A B C D, then diverges:
Logical sequence B:  A B C D | X Y
Block table B:       0 -> physical block 7   (shared prefix)
                     1 -> physical block 9   (new block)
```

For block size `P` and sequence length `S`, the allocator needs `ceil(S/P)` blocks. Only the final block can be partially filled, so tail slack is bounded by `P - 1` token slots per sequence. Free physical blocks can be used without finding a large contiguous span.

In practice, the attention kernel uses the block table to locate the K/V blocks needed by each query. The runtime can share immutable prefix blocks among compatible requests; after requests diverge, they allocate different blocks. Parallel sampling and beam-like branching can therefore avoid copying a common prefix. Implementations use block ownership/reference tracking and may use copy-on-write when a shared block must be modified.

**What paging does and does not do:**

- It improves allocation and reduces fragmentation; it does not reduce the bytes required by the live K/V values.
- It supports variable-length requests and prefix sharing; it does not make unlike prompts share state.
- It borrows the logical-to-physical mapping idea from virtual memory, but PagedAttention is not necessarily swapping KV pages to disk. The key mechanism is block allocation and block-aware attention in accelerator memory.
- Smaller blocks reduce tail slack and make sharing more granular, but increase block-table and scheduling overhead. Larger blocks reduce metadata/lookup work but can leave more unused capacity in the last block.

The original vLLM work combines PagedAttention with a serving system; PagedAttention is the attention/cache design, not the whole server. NVIDIA and other runtimes have related paged-KV implementations with their own details.

## Prefix sharing: avoid duplicate prefill state

When requests have an identical token prefix under compatible model and execution settings, a runtime can sometimes reuse the prefix's computed KV state instead of repeating prefill. With paged storage, the requests can reference the same physical prefix blocks and allocate separate blocks once their token sequences diverge.

Reuse is conditional: token IDs and relevant model weights, adapters, positional settings, cache dtype, and attention semantics must be compatible. Semantically similar but differently tokenized prompts are not automatically interchangeable. Prefix caching is different from caching a completed HTTP response and consumes memory that could otherwise serve other active requests.

## KV-cache compression: reduce representation or retained state

Compression targets the size or count of live K/V values. Unlike paging, which improves how state is allocated, compression can change the stored representation or what context remains available.

### 1. Quantize K and V

KV quantization maps higher-precision values to a lower-bit representation, usually with scale metadata so the approximate values can be reconstructed or used in a low-precision kernel. The ideal storage ratio is approximately `target bits / original bits`, before scales, alignment, packing, and metadata. For example, 16-bit to 8-bit storage halves the raw value payload; 16-bit to 4-bit quarters it.

The practical result depends on the quantization granularity (for example, groups or channels), treatment of outliers, K-versus-V distributions, available kernels, and how frequently values must be dequantized. Fewer bits do not guarantee lower latency if conversion or unsupported kernels add overhead. Test long-context retrieval and generation quality as well as memory and latency.

### 2. Evict or select tokens

An eviction policy removes some positions' K/V state to cap cache growth. Policies can retain a recent sliding window, preserve selected initial/system tokens, retain tokens estimated to be important, or combine a fixed recent window with a smaller selected set.

The policy is an approximation to the model's full-context attention behavior. A token that looks unimportant at one step, layer, or head may matter later. Eviction can therefore harm retrieval of distant facts or exact long-context behavior. It is safest when the model was designed for a bounded window or the task tolerates forgetting; otherwise evaluate task-specific quality carefully.

### 3. Share K/V across attention heads

Multi-query attention (MQA) and grouped-query attention (GQA) reduce the number of K/V heads relative to query heads. Because the cache stores K and V per KV head, fewer KV heads lower cache storage and decode-time K/V reads.

This is an architectural design, not a general post-hoc compression pass over an arbitrary model. See [MQA and GQA](attention-kernels-and-model-architectures.md#multi-query-and-grouped-query-attention-reduce-kv-heads) for their layouts and quality trade-offs.

### 4. Store a lower-dimensional latent

Low-rank or latent-cache methods represent K/V state using a smaller learned or structured representation rather than retaining every original vector directly. The runtime/model reconstructs or consumes that representation during attention. This can reduce the state dimension, but adds projection/reconstruction work and often requires model-specific training or conversion. The exact method varies; a generic low-rank factorization is not automatically a drop-in replacement for a pretrained model's K/V cache.

## Compare the memory levers

| Method | Changes | State fidelity | Main benefit | Main cost |
| --- | --- | --- | --- | --- |
| Paged KV allocation / PagedAttention | Physical allocation and address mapping | Exact values retained | Less fragmentation; flexible allocation and sharing | Block-table/allocator work; last-block slack |
| Prefix caching | Reuses compatible prefix blocks | Exact for compatible state | Less repeated prefill | Cache capacity and compatibility constraints |
| KV quantization | Numeric precision of K/V | Approximate | Fewer bytes and potentially less memory traffic | Quantization error, metadata, kernel/dequant overhead |
| Token eviction/selection | Which token states remain | Approximate unless model semantics are windowed | Bounded memory as context grows | Forgotten context can harm quality |
| Low-rank/latent cache | Representation dimension | Approximate or model-specific | Smaller state representation | Extra compute and compatibility/training requirements |
| MQA/GQA | Model's K/V-head count | Model-defined | Smaller cache and K/V traffic | Requires an appropriate model architecture |
| FlashAttention | How attention is computed and data moved | Exact attention math (floating-point order can differ) | Less intermediate attention IO | Does not reduce persistent KV-state size |

These approaches can be combined when the runtime and model support them. For example, a system can allocate quantized KV in pages, share compatible prefix pages, and execute attention with an optimized kernel. They solve different parts of the memory/performance problem.

## Keep these mechanisms distinct

| Mechanism | What changes | Main benefit | What it does not do |
| --- | --- | --- | --- |
| Paged KV allocation | Where live KV blocks are placed and when they are allocated | Less fragmentation/reservation waste; easier sharing | Does not reduce the bytes in each K/V value |
| Prefix caching | Reuse of compatible already-computed prefix state | Avoid repeated prefill on shared prefixes | Does not help unrelated prefixes |
| KV quantization | Numeric precision of cached K/V | Fewer bytes and less memory traffic | Does not make the cache's token count smaller |
| Eviction / low-rank compression | Which state or representation is retained | Fits more context/requests in memory | May change model behavior or require extra work |
| FlashAttention | How attention math is tiled and moved through GPU memory | Less high-bandwidth-memory traffic for attention | Does not eliminate the KV cache |
| MQA/GQA | Model's number of KV heads | Smaller KV state and decode traffic | Is not an allocator or cache-paging scheme |

## When memory becomes the bottleneck

Estimate memory at the expected **active** context length and concurrency, then add model weights, temporary activations/workspaces, runtime buffers, and a safety margin. Watch how allocated KV blocks, active tokens, preemption/recomputation, and request queueing change under load. A model that fits on one accelerator for one request may not fit at the target concurrency.

If KV state fits but attention kernels spend excessive time moving intermediate data, read [Attention kernels and model architectures](attention-kernels-and-model-architectures.md). If a new prompt stalls active generations, investigate [scheduling](scheduling-caching-and-speculation.md) or [prefill–decode disaggregation](prefill-decode-disaggregation.md), rather than assuming compression is the answer.

## References

- Amit Shekhar, [KV Cache in LLMs](https://outcomeschool.com/blog/kv-cache-in-llms), [KV Cache Compression](https://outcomeschool.com/blog/kv-cache-compression), and [Paged Attention in LLMs](https://outcomeschool.com/blog/paged-attention-in-llms).
- Amit Shekhar, [How does Model Quantization work?](https://outcomeschool.com/blog/how-does-model-quantization-work).
- Woosuk Kwon et al., [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/abs/2309.06180), SOSP 2023.
- vLLM, [PagedAttention design overview](https://blog.vllm.ai/2023/06/20/vllm.html) and [current documentation](https://docs.vllm.ai/en/stable/).
- JAX Scaling Book, [Inference](https://jax-ml.github.io/scaling-book/inference/) chapter, including its discussion of sharding the KV cache. The specific layouts and hardware examples are implementation-dependent.
- See also [Grouped-Query Attention](attention-kernels-and-model-architectures.md#multi-query-and-grouped-query-attention-reduce-kv-heads) and [FlashAttention](attention-kernels-and-model-architectures.md#flashattention-reduce-data-movement).

Sources accessed 2026-10-30.
