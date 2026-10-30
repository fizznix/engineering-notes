# Inference Engineering

[Back to the repository index](../README.md)

Inference engineering is the work of turning a trained model into a responsive, reliable, and cost-effective service. The challenge is not just making a model run: it is managing its computation, memory, scheduling, hardware, and user-visible latency under real workloads.

## Start here

Use the [learning map](MAP.md) for a guided sequence, prerequisite concepts, and paths through the notes. If you are new to LLMs, first skim the [Generative AI index](../06-generative-ai/README.md) for the model concepts that inference builds on.

## Notes

1. [Generation lifecycle and performance metrics](generation-lifecycle-and-metrics.md) — autoregressive generation, prefill/decode, idealized compute/memory bounds, and benchmark methodology.
2. [Hardware and GPU kernels](hardware-and-kernels.md) — accelerator memory and execution, CUDA kernels, GPUs, TPUs, and Groq's LPU.
3. [KV cache and memory management](kv-cache-and-memory.md) — why the KV cache exists, how it grows, and detailed paging, sharding, sharing, quantization, eviction, and low-rank trade-offs.
4. [Attention kernels and model architectures](attention-kernels-and-model-architectures.md) — FlashAttention's tiling and online softmax, MQA/GQA, and MoE as distinct ways to change compute or memory costs.
5. [Scheduling, caching, and speculative decoding](scheduling-caching-and-speculation.md) — batching, continuous scheduling, prefix reuse, token streaming, and draft-and-verify methods.
6. [Architecture choices, routing, and distillation](architecture-routing-and-distillation.md) — SLMs, model/mixture routing, MoE, and teacher–student compression.
7. [Prefill–decode disaggregation](prefill-decode-disaggregation.md) — when separating the two phases helps, and the cost of moving KV state between workers.
8. [Serving engines and model formats](serving-engines-and-model-formats.md) — software-stack boundaries, engines and orchestration, model quantization, multi-GPU parallelism, file formats, and local runners.

## Related domains

[Machine learning](../05-machine-learning/README.md) · [Generative AI](../06-generative-ai/README.md) · [Cloud and DevOps](../10-cloud-devops/README.md) · [Projects and labs](../13-projects-labs/README.md)

## Source collection

The sequence covers all six parts and 28 topic entries in the [LLM Inference Engineering repository](https://github.com/amitshekhariitbhu/llm-inference-engineering), along with the two supplied prefill articles, their relevant linked explainers, Inference Engineering's software chapter, and the JAX Scaling Book's inference/roofline chapters. See the [coverage table in the learning map](MAP.md#coverage-of-the-source-series) and its supplemental-source note. Repeated explanations are consolidated; the notes add derivations, examples, distinctions, and further reading rather than reproducing the articles.

The source articles link to an Outcome School GPU explainer that returned 404 when checked on 2026-10-30. For GPU execution and memory background, these notes use NVIDIA's maintained [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/) instead.
