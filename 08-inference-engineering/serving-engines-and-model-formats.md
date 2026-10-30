# Serving Engines and Model Formats

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

An inference deployment has several layers: model weights and metadata, an execution/runtime engine, a serving API and scheduler, and the hardware/operations around it. Tools may span multiple layers, but the distinctions matter when diagnosing compatibility, performance, or deployment behavior.

## Serving runtimes and engines

An inference engine loads a model, selects kernels, manages memory and requests, and executes generation. A production serving system may also expose APIs, stream responses, batch requests, collect metrics, and coordinate multiple workers.

Examples from the source collection:

- **vLLM** is an open-source serving/inference system. Its PagedAttention paper introduced block-based KV management for efficient high-throughput serving; its current system combines a runtime, scheduling, memory management, and multiple execution backends. Its supported architectures and features continue to evolve.
- **SGLang** is an open-source serving framework. Its RadixAttention work organizes reusable prefixes for cache lookup and sharing, alongside scheduling and structured-generation features. Capabilities and supported hardware evolve by release.
- **TensorRT-LLM** is NVIDIA's toolkit/runtime stack for optimized LLM execution and serving on supported NVIDIA hardware. Its workflows can transform model graphs, apply hardware-specific kernels and precision choices, and build/deploy engines; serving features and model support depend on the exact release and device.

These are not interchangeable by name alone. Compare model and operator support, quantization formats, parallelism, cache behavior, APIs, observability, deployment requirements, and performance on your workload. Feature lists change quickly; use the official documentation for the version being tested.

## How serving software is layered

A useful mental model separates device execution, model frameworks, per-model serving, and fleet-level coordination:

```text
accelerator + device runtime/libraries
              ↓
framework and compiler
              ↓
inference engine / worker
              ↓
orchestration across workers and replicas
              ↓
application API, ingress, and deployment platform
```

- **Device runtime and libraries** expose kernels, memory operations, graphs, and tuned primitives. CUDA and libraries such as cuBLAS are NVIDIA-specific examples; other accelerators have their own runtimes and libraries.
- **Frameworks and compilers** represent model operations and transform or compile them for execution. PyTorch with `torch.compile` and JAX with XLA are examples. Model serialization formats such as Safetensors and ONNX are related to this layer, but are not serving engines.
- **Inference engines** load and execute models, select supported kernels, manage per-worker scheduling and KV state, and often provide a serving API. vLLM, SGLang, and TensorRT-LLM are examples.
- **Orchestration** coordinates multiple engine workers or replicas: request routing, scaling, cache locality, and possibly separate prefill and decode pools. NVIDIA [Dynamo](https://docs.nvidia.com/dynamo/latest/) is one distributed-serving example; its integrations and capabilities vary by release.
- **Application ingress and deployment** can add API gateways, authentication, rate limiting, service discovery, and infrastructure autoscaling. These responsibilities may be provided by a proxy, Kubernetes components, the orchestration layer, or a combination.

The boundaries are conceptual rather than strict. An engine can expose an API and manage multiple local workers; an orchestration system may also provide routing or deployment features. The distinction is still useful when debugging: an engine runs model work, while fleet orchestration decides where that work should run and how worker groups coordinate. See [prefill–decode disaggregation](prefill-decode-disaggregation.md) for a case where coordination also moves KV state between pools.

## Model files are not serving engines

### GGUF

**GGUF** is a binary model-file format used by GGML-based inference executors. It packages tensor data and metadata intended to make model files self-describing and practical to load; files may encode weights in different precisions/quantizations. GGUF is not itself a server or a guarantee that every runner supports the architecture, tensor encoding, or model feature.

### Ollama

**Ollama** is a tool for running and managing models locally and exposing an application interface. It handles model acquisition/configuration and invokes a suitable inference backend; it is not synonymous with GGUF. Confirm the current backend, model format, accelerator path, context configuration, and supported features for the Ollama release in use. Its documentation also covers cloud-model workflows, so "Ollama" should not be treated as meaning local-only in every deployment.

## Model quantization: representation, kernels, and quality

### How low-bit representations work

Quantization maps a set of higher-precision values onto a smaller set of representable values. A common affine form is:

```text
x ≈ scale × (q - zero_point)
```

`q` is an integer code; `scale` and `zero_point` describe how that code maps back to an approximate real value. Symmetric quantization typically uses a zero point of zero; asymmetric schemes can represent ranges that are not centered around zero. Scales may be shared across a tensor, channel, group, or other block. Finer-grained scales can reduce error but require metadata and extra work.

FP16 and BF16 use 16-bit floating-point values; FP8 uses 8-bit floating point. INT8 and INT4 use 8- and 4-bit integer codes. The smaller code range lowers storage but makes scale selection and outlier handling more important. Some formats pack multiple low-bit values into machine words and store additional scale/zero metadata.

### Estimate weight storage

For `P` parameters stored at `b` bits each, raw weight payload is approximately:

```text
weight bytes ≈ P × b / 8
```

For 7 billion parameters, the idealized weight payload is:

| Representation | Approximate raw payload |
| --- | ---: |
| 16-bit | 14.0 billion bytes (about 13.0 GiB) |
| 8-bit | 7.0 billion bytes (about 6.5 GiB) |
| 4-bit | 3.5 billion bytes (about 3.3 GiB) |

These are payload estimates, not total runtime memory. Real files/runtimes add scales, zero points, padding, alignment, metadata, temporary buffers, KV cache, and allocator overhead. The model's total memory can be much larger than its weight file.

### Quantization families and what they change

- **Weight-only quantization:** store weights at lower precision, then use a kernel that reads/dequantizes or directly computes with the packed representation. It can reduce weight traffic, especially in decode, but conversion/kernel efficiency matters.
- **Weight-and-activation quantization:** quantize both weights and intermediate activations. It can use low-precision matrix instructions efficiently but requires supported ranges and kernels for activations as well.
- **Post-training quantization (PTQ):** quantize a trained model without a full retraining run; calibration or optimization uses representative data to reduce error.
- **Quantization-aware training (QAT):** simulate quantization effects during training/fine-tuning so the model can adapt to them.
- **Method/format examples:** GPTQ and AWQ are weight-quantization approaches; GGUF is a container format that can store tensors in different quantized encodings; an execution library/runtime still needs support for the exact tensor type and kernel.

Quantization does not necessarily make every operation lower precision. A runtime may keep some layers or intermediate operations at higher precision, and some quantized weights are unpacked or dequantized for computation.

### Keep the three quantization targets separate

| Target | What is stored/computed differently | Main memory affected |
| --- | --- | --- |
| Model-weight quantization | Learned parameter tensors | Persistent model weight memory and weight-read traffic |
| KV-cache quantization | Per-request attention keys and values | Growing sequence-state memory and KV-read traffic |
| Activation/intermediate quantization | Temporary hidden states and operation inputs/outputs | Workspace/intermediate memory and matrix-kernel execution |

Weight quantization and KV quantization can be combined, but their error sources and kernels differ. See [KV-cache quantization](kv-cache-and-memory.md#1-quantize-k-and-v) for the cache-specific case.

### Evaluate, do not infer speed from bit width

Fewer bits can allow more concurrent requests or larger models and can reduce memory traffic. Actual latency may improve, stay similar, or worsen depending on hardware support, kernel availability, dequantization overhead, batch size, and whether the workload was compute- or bandwidth-bound. Quality can shift, especially in difficult or long-context tasks.

Test the exact model checkpoint, tokenizer, quantized artifact, runtime version, and accelerator. Record quality against a baseline plus TTFT, inter-token latency, throughput, memory, and cost. A label such as "4-bit" alone does not specify group size, scales, quantization algorithm, or runtime behavior.

## Parallel inference across accelerators

Engines can split work across devices when a model or KV state exceeds one device's capacity, or to increase throughput. Common choices are tensor parallelism (shard layer operations), pipeline parallelism (place layer ranges on different devices), data parallelism (replicate the model for separate requests), and expert parallelism (distribute MoE experts). Their communication costs and trade-offs are described in [Hardware and GPU kernels](hardware-and-kernels.md#scale-one-model-across-accelerators).

## Architecture affects runtime needs

## Architecture affects runtime needs

An MoE model may have routing and expert communication requirements; a GQA model has a different KV-head layout from MHA; multimodal models may add encoders and non-text state. Check the serving engine's exact architecture path rather than assuming that a model loads merely because a related model family is listed.

## A stack-selection checklist

1. **Model compatibility:** architecture, tokenizer, adapters, multimodal components, and generation features.
2. **Hardware fit:** accelerator family, memory capacity/bandwidth, kernels, and host/interconnect needs.
3. **Request behavior:** streaming, batching, prefix reuse, speculative methods, structured output, and phase separation.
4. **Operations:** API compatibility, observability, rollout/versioning, failure recovery, and deployment model.
5. **Evidence:** compare supported release configurations with measured TTFT, inter-token latency, throughput, memory, quality, and cost for a representative workload.

## References

- Amit Shekhar, [How does vLLM work?](https://outcomeschool.com/blog/how-does-vllm-work), [How does SGLang work?](https://outcomeschool.com/blog/how-does-sglang-work), [How does TensorRT-LLM work?](https://outcomeschool.com/blog/how-does-tensorrt-llm-work), [How does GGUF work?](https://outcomeschool.com/blog/how-does-gguf-work), and [How does Ollama work?](https://outcomeschool.com/blog/how-does-ollama-work).
- Inference Engineering, [Software](https://inferenceengineering.tech/chapters/software/) chapter. Its stack diagram is a useful taxonomy; tool rankings and feature lists should be checked against current versions and representative benchmarks.
- vLLM, [official documentation](https://docs.vllm.ai/en/stable/) and [PagedAttention paper](https://arxiv.org/abs/2309.06180).
- SGLang, [official documentation](https://docs.sglang.io/).
- NVIDIA, [TensorRT-LLM documentation](https://nvidia.github.io/TensorRT-LLM/).
- NVIDIA, [Dynamo documentation](https://docs.nvidia.com/dynamo/latest/) and [source repository](https://github.com/ai-dynamo/dynamo).
- GGML project, [GGUF specification](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md).
- Ollama, [official documentation](https://docs.ollama.com/).
- Amit Shekhar, [How does Model Quantization work?](https://outcomeschool.com/blog/how-does-model-quantization-work).

Sources accessed 2026-10-30.
