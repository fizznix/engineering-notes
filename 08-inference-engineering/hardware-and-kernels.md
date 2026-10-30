# Hardware and GPU Kernels

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

Inference performance depends on how quickly the model's arithmetic can be performed **and** how quickly weights, activations, and KV state can reach the compute units. GPU, TPU, and LPU designs make different trade-offs; the right choice depends on model support, workload shape, software, memory, networking, and measured end-to-end behavior.

## The accelerator memory hierarchy

Accelerators have several levels of storage, with different size, speed, and sharing characteristics. A simplified NVIDIA GPU view is:

```text
host RAM  <--- host/device transfers --->  GPU HBM (large, off-chip)
                                               |
                                     L2 / per-SM caches
                                               |
                                  registers + shared memory
                                               |
                                     arithmetic units
```

Registers and shared memory are small and close to compute. HBM holds model weights and large KV state but takes more time and energy to access. Caches and data movement connect these levels. Exact names, sizes, and organization differ across accelerator generations.

A useful first diagnostic is **arithmetic intensity**:

```text
arithmetic intensity = arithmetic operations / bytes moved
```

Low-intensity work is more likely to be bandwidth-bound; high-intensity work can make better use of peak arithmetic throughput. A roofline-style upper bound is the lesser of compute throughput and `memory bandwidth × arithmetic intensity`, before accounting for launch overhead, synchronization, communication, and other constraints.

This explains why prefill, with many input positions processed together, often makes large matrix operations efficient, while a small decode batch can be sensitive to repeatedly reading model weights and KV state. Batching may reuse weight reads across sequences and change that balance. See [generation phases and metrics](generation-lifecycle-and-metrics.md).

## What a CUDA kernel does

A **kernel** is device code launched by the CPU (host) to run across many GPU threads. CUDA organizes work into a grid of thread blocks; hardware schedules those blocks onto multiprocessors (SMs), which execute threads in warps. Threads in one block can cooperate through shared memory and synchronization. Blocks are generally independent unless coordination is arranged through additional mechanisms.

For a one-dimensional elementwise operation, each thread can compute its global element index:

```cpp
int i = blockIdx.x * blockDim.x + threadIdx.x;
if (i < n) {
    out[i] = a[i] + b[i];
}
```

The host allocates/copies device data as needed, launches the grid, and later consumes results. GPU execution is asynchronous relative to many host operations, so synchronization and timing must be measured correctly. Frequent small launches or host/device copies can erase the benefit of a fast kernel.

### What makes kernels fast or slow

- **Coalesced access:** neighboring threads access neighboring addresses so memory transactions serve many threads.
- **Reuse and tiling:** load a data tile into registers/shared memory and reuse it before fetching another tile. Matrix-multiplication kernels do this to increase arithmetic per byte transferred.
- **Occupancy and resource pressure:** registers/shared memory allow fast reuse, but using too much per block can reduce the number of blocks active on an SM.
- **Launch and synchronization overhead:** a tiny operation may cost more to launch than to compute.
- **Numerical precision:** lower-precision arithmetic can increase throughput, but changes rounding and requires supported instructions/kernels.

### Kernel fusion and CUDA graphs

**Kernel fusion** combines adjacent operations so an intermediate need not be written to and reread from HBM, and can reduce launch count. It is not automatically beneficial: a fused kernel can use more registers, lower occupancy, or be less specialized for each stage.

**CUDA graphs** capture and replay a sequence of GPU operations, which can reduce repeated CPU-side launch overhead in suitable workloads. Graph capture/replay has shape, memory, and operation compatibility requirements; dynamic sequence lengths and scheduling need runtime-specific handling.

Custom attention kernels and optimized GEMM/MoE kernels are specialized implementations of these ideas. [FlashAttention](attention-kernels-and-model-architectures.md#flashattention-reduce-data-movement) is a concrete example of tiling plus IO-aware softmax for attention.

## GPUs

GPUs expose a general parallel execution model useful for many workloads. For inference, practical performance depends on more than advertised FLOPs: memory bandwidth/capacity, kernel support for the model's operations and precision, batch/context shape, host scheduling, and inter-GPU connectivity all matter.

The source repository's Outcome School GPU explainer, `https://outcomeschool.com/blog/how-does-a-gpu-work-for-deep-learning`, returned 404 when checked on 2026-10-30. NVIDIA's maintained [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/) is used here for CUDA execution concepts; it describes NVIDIA GPUs, not every vendor's GPU.

## Google TPUs

Google TPUs are Google-designed ASICs for machine-learning workloads. Their matrix units use systolic-array-style data movement: values flow through connected multiply-accumulate units so matrix operations can reuse operands without repeatedly returning to large memory. Cloud TPU execution is compiler-oriented; XLA compiles supported framework computations to TPU code, while the host handles other work.

This specialization can suit workloads dominated by supported matrix operations and shapes. Model operators, dynamic shapes, layout, host input/output, software framework, and available TPU topology affect actual suitability. A TPU is not simply a GPU with a different name, and a peak matrix-unit number is not an end-to-end inference benchmark.

## LPUs

**LPU (Language Processing Unit)** is Groq's product term for its inference-focused accelerator family; it is not a generic standards-defined accelerator class. The source explainer describes a design emphasizing on-chip SRAM, compiler-scheduled execution, and a network that connects many chips. Those are useful architectural ideas to compare, but precise capabilities, memory capacity, supported models, and performance are product- and software-version-specific.

Large on-chip memory can reduce movement to external memory for workloads that fit the design, while compiler scheduling and a scale-out interconnect target predictable execution. Trade-offs include capacity/model fit, supported operations and formats, batching behavior, programming/runtime ecosystem, and availability. Treat vendor throughput figures as workload-specific; compare identical model, precision, prompt/output distribution, concurrency, and latency targets.

## Scale one model across accelerators

When weights or KV state do not fit on one device, or more throughput is needed, serving stacks can partition work:

- **Tensor parallelism** shards parts of layer operations across devices. It can distribute weight/memory demand, but introduces collectives between devices, often repeatedly through layers.
- **Pipeline parallelism** places different layer ranges on different devices. It can fit a deeper model across devices, but introduces stage-to-stage transfers and pipeline bubbles, especially for small or low-concurrency workloads.
- **Data parallelism** runs replicas and sends independent requests to them. It can raise request throughput when each replica fits, but does not reduce the per-replica model memory.
- **Expert parallelism** distributes MoE experts and routes tokens to their devices, making dispatch and all-to-all communication part of the cost.

Parallelism helps only if the capacity/throughput gain exceeds communication and scheduling overhead. Interconnect bandwidth and topology matter as much as the number of accelerators.

## Compare hardware fairly

Benchmark with the same model/checkpoint, tokenizer, precision, context and output lengths, decoding settings, concurrency/arrival pattern, and API behavior. Report TTFT and inter-token latency percentiles as well as end-to-end latency, input/output throughput, memory use, quality, and cost. A vendor's maximum tokens-per-second number is not comparable if its batch, output length, or latency target differs.

## References

- Amit Shekhar, [How do CUDA Kernels work?](https://outcomeschool.com/blog/how-do-cuda-kernels-work), [How does a Google TPU work?](https://outcomeschool.com/blog/how-does-a-google-tpu-work), and [How does an LPU work?](https://outcomeschool.com/blog/how-does-an-lpu-work).
- NVIDIA, [CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-programming-guide/).
- Google Cloud, [Introduction to Cloud TPU](https://docs.cloud.google.com/tpu/docs/intro-to-tpu) and [TPU architecture](https://docs.cloud.google.com/tpu/docs/system-architecture-tpu-vm).
- Groq, [Groq platform](https://groq.com/platform) and [developer documentation](https://console.groq.com/docs/overview).
- See also [FlashAttention](attention-kernels-and-model-architectures.md#flashattention-reduce-data-movement) and [serving engines and model formats](serving-engines-and-model-formats.md).

Sources accessed 2026-10-30.
