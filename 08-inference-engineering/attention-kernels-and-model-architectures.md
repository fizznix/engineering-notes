# Attention Kernels and Model Architectures

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

Inference cost depends both on **what the model computes** and **how the runtime executes it**. Attention kernels, attention-head layout, and sparse model architectures act at different levels. Keeping those levels separate makes it easier to choose an optimization and understand its quality and hardware trade-offs.

## A short attention refresher

For query, key, and value matrices `Q`, `K`, and `V`, scaled dot-product attention is:

```text
Attention(Q, K, V) = softmax(Q Kᵀ / √dₖ) V
```

The scores compare queries with keys; the normalized scores weight the values. In causal language modeling, a position cannot attend to future positions. During prefill, the model computes attention over many prompt positions. During decode, a new query attends to retained keys and values from the prefix and generated tokens.

For definitions and the full request timeline, see [Generation lifecycle and performance metrics](generation-lifecycle-and-metrics.md).

## FlashAttention: reduce data movement

For a sequence of length `N`, a straightforward implementation forms the score matrix `S = QKᵀ`, applies row-wise softmax, then multiplies by `V`. The dense attention arithmetic remains quadratic in sequence length. The problem is also **IO**: writing and rereading an `N × N` score/probability matrix in high-bandwidth memory (HBM) can cost more than keeping the arithmetic units busy.

**FlashAttention** is an IO-aware exact-attention algorithm. It tiles the work so smaller query/key/value blocks fit in fast on-chip memory (often described as SRAM), computes partial scores there, and streams through key/value blocks. It writes the final attention output without materializing the full `N × N` matrix in HBM.

### Online softmax, one query tile at a time

Softmax normally depends on all scores in a row, which seems to prevent processing only one key tile at a time. FlashAttention maintains a numerically stable running maximum `m`, a running denominator `l`, and an unnormalized weighted-value sum `u` for each query row:

```text
Initialize: m = -∞, l = 0, u = 0

For each key/value tile:
    S = Q_tile K_tileᵀ / √dₖ + causal_mask
    m_new = max(m, row_max(S))
    P = exp(S - m_new)
    α = exp(m - m_new)
    l_new = α·l + row_sum(P)
    u_new = α·u + P·V_tile
    m, l, u = m_new, l_new, u_new

Final output for the query tile: u / l
```

If a later tile contains a larger score, `m_new` changes; `α` rescales the accumulated denominator and output so they remain in the same normalization frame. The final `u/l` is the same normalized weighted sum as a full-row softmax in exact arithmetic. Causal masking sets disallowed future positions to negative infinity before exponentiation.

The benefit is avoiding large intermediate HBM writes and rereads. FlashAttention still performs dense attention's quadratic arithmetic; it is not a subquadratic/linear-attention approximation. Its temporary storage is much smaller than materializing all scores, but query/key/value tiles still move through the memory hierarchy.

### Variants and where they fit

- **FlashAttention (v1):** introduced the tiled, IO-aware exact-attention algorithm and online softmax.
- **FlashAttention-2:** improves GPU work partitioning and parallelism, including how work is distributed across thread blocks and warps; it retains the exact-attention approach.
- **FlashAttention-3:** targets Hopper-generation GPU features with asynchronous data movement/computation and low-precision support such as FP8. It is hardware-specific; do not assume its gains or availability transfer to another accelerator.

These versions are kernel implementations, not model architectures. In inference, prefill has many query positions and often maps naturally to tiled attention. Decode may have only one or a few new query positions per sequence and can be constrained by KV reads, so specialized decode kernels may be preferable. Select based on runtime, hardware, head dimensions, context lengths, and batching.

### What FlashAttention does not do

| It does | It does not |
| --- | --- |
| Reduce intermediate attention-matrix IO and storage | Eliminate dense attention's quadratic arithmetic |
| Compute the same mathematical attention using tiling/online normalization | Compress or evict the persistent KV cache |
| Complement paged KV layouts when an implementation supports both | Replace PagedAttention or guarantee compatibility with every kernel/runtime |
| Often help long-sequence prefill | Guarantee faster decode for every batch and sequence shape |

For the separation between attention-kernel IO and persistent cache allocation, see [PagedAttention and KV compression](kv-cache-and-memory.md#pagedattention-allocate-kv-state-as-blocks).

## Multi-query and grouped-query attention: reduce KV heads

In ordinary multi-head attention (MHA), each query head has its own key and value head. During autoregressive inference, retaining K/V for every head can be expensive.

- **Multi-query attention (MQA):** many query heads share one K/V head. This can substantially reduce KV-cache size and K/V bandwidth, but may trade off model quality.
- **Grouped-query attention (GQA):** query heads are divided into groups, and heads within a group share K/V. It sits between MHA and MQA: more KV heads than MQA, fewer than MHA.

If `Hq` is the number of query heads and `Hkv` the number of KV heads, K/V cache storage scales with `Hkv`. GQA is a model architecture (and typically a trained/uptrained model property); a serving runtime cannot simply turn an arbitrary MHA checkpoint into equivalent GQA without a model conversion/training procedure and quality validation.

For example, with 32 query heads and 8 KV heads, each KV head serves a group of four query heads. With the same layer count, head dimension, and cache precision, that layout stores one quarter as many K/V head vectors as MHA with 32 KV heads. MQA takes the ratio further to one KV head shared by all 32 queries. The actual model's quality and speed still depend on training, kernel implementation, and hardware; this storage ratio is not itself a measured speedup.

| Layout | Query heads | KV heads | K/V sharing |
| --- | ---: | ---: | --- |
| MHA | `Hq` | `Hq` | One K/V head per query head |
| GQA | `Hq` | Between 1 and `Hq` | Each K/V head serves a group of query heads |
| MQA | `Hq` | 1 | All query heads share one K/V head |

## Mixture of Experts: sparse parameter activation

A **Mixture-of-Experts (MoE)** layer contains multiple feed-forward expert networks and a router. For each token, the router selects a subset of experts (often called top-k routing); only those experts perform that token's expert computation.

This can increase total parameter capacity without executing every expert for every token. It does not imply that all unused expert weights disappear from memory: model placement, expert parallelism, offloading, and routing determine memory and communication cost. Expert imbalance, all-to-all traffic, and dispatch overhead can erode gains. Measure active computation and total resident memory separately.

## Match the lever to the bottleneck

| Lever | Main target | Key trade-off / limit |
| --- | --- | --- |
| FlashAttention and related kernels | Attention IO and intermediate memory | Hardware-, kernel-, and sequence-shape-dependent; KV cache remains |
| GQA/MQA | KV cache size and K/V traffic | Requires compatible model architecture; MQA may affect quality |
| MoE | Compute per token relative to total model capacity | Router and inter-device communication; all expert weights may still need placement |
| KV paging / quantization | KV allocation or representation | These are cache-management choices, covered in [KV cache and memory management](kv-cache-and-memory.md) |

No technique is a universal "make inference faster" switch. A compute-heavy prefill can benefit from efficient tiled kernels; a memory-constrained decode workload may benefit more from fewer KV heads, smaller KV values, or higher effective bandwidth.

## References

- Amit Shekhar, [Decoding Flash Attention in LLMs](https://outcomeschool.com/blog/decoding-flash-attention), [Grouped Query Attention](https://outcomeschool.com/blog/grouped-query-attention), and [Mixture of Experts Explained](https://outcomeschool.com/blog/mixture-of-experts).
- Tri Dao et al., [FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135).
- Tri Dao, [FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning](https://arxiv.org/abs/2307.08691) and [FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision](https://arxiv.org/abs/2407.08608).
- Joshua Ainslie et al., [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245).
- NVIDIA, [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/), for GPU execution and memory hierarchy.

Sources accessed 2026-10-30.
