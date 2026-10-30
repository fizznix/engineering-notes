# Architecture Choices, Model Routing, and Distillation

[Back to the learning map](MAP.md) · [Back to Inference Engineering](README.md)

Inference efficiency can come from choosing a smaller model, selecting a model per request, activating only part of a model, or training a smaller student to imitate a larger teacher. These approaches happen at different times and have different correctness and operations implications.

## Small language models

**Small language model (SLM)** is a relative label, not a universal parameter-count boundary. A model is small for a workload when its quality, memory footprint, latency, and energy fit the target device and service better than larger alternatives. Some articles use a convenient parameter threshold, but device capability and task complexity make the boundary move.

Why consider an SLM?

- Lower parameter memory and often lower compute per generated token.
- More feasible local or on-device inference, which can reduce network dependence and keep some data on the device.
- Less serving cost and potentially lower latency when the model fits in memory and its kernels are efficient.

The trade-offs are task coverage, reliability on difficult prompts, context length, tool use, multilingual or multimodal capability, and quality. "Smaller" does not guarantee faster: unsupported operators, poor quantized kernels, CPU/GPU transfers, or batch behavior can dominate.

An unquantized model's raw weight payload is approximately:

```text
weight bytes ≈ parameter count × bytes per parameter
```

For example, 7 billion parameters at 16 bits (2 bytes each) require about 14 billion bytes, or 13.0 GiB, just for raw weights. Runtime workspaces, KV cache, metadata, and allocator overhead are additional. See [model and KV quantization](serving-engines-and-model-formats.md#model-quantization-representation-kernels-and-quality) for how lower-bit weights change this estimate.

## Knowledge distillation

**Distillation** trains a student model to reproduce useful behavior from a teacher model. The teacher can provide probability distributions over next tokens (or classes), generated examples, rationales/targets where appropriate, or a mixture of these. At inference time, the student is deployed without needing to call the teacher on every request.

### Hard targets and soft targets

A hard label might say only "the correct class is cat." The teacher's distribution can also indicate that "dog" is more plausible than "car." That relative probability information is a **soft target**.

If teacher and student logits are `z_t` and `z_s`, a common soft-target construction is:

```text
p_t = softmax(z_t / T)
p_s = softmax(z_s / T)
```

Temperature `T > 1` softens the distributions and can expose relative preferences that are nearly invisible in a very sharp distribution. A common training objective combines a distillation loss with the original task loss:

```text
L = α · T² · KL(p_t || p_s) + (1 - α) · CE(y, p_s at task temperature)
```

`KL` is a divergence between teacher and student distributions; `CE` is cross-entropy against the ground-truth target `y`. The `T²` factor is commonly used to compensate for temperature's effect on gradient scale. The exact objective, temperature, data, and weighting depend on the task and distillation method. The softened temperature is a training device; the deployed student uses its normal inference-time decoding configuration.

Distillation compresses behavior into a student, not the teacher's parameters byte-for-byte. The student can fail on cases the teacher handled, inherit teacher mistakes or biases, and need task-specific evaluation. Knowledge distillation is an offline model-training approach; it is not the same as speculative decoding, where a draft proposes tokens and a target verifies them at request time.

## Mixture of Experts

An MoE model replaces some dense feed-forward layers with a set of **experts** and a learned router. For each token representation `x`, the router selects a subset `E(x)` (often top-k), and the layer combines those experts' outputs:

```text
y = Σ(e in E(x)) gate_e(x) · expert_e(x)
```

Only selected experts perform that token's expert computation, so active compute can be much smaller than if every expert ran for every token. However, total parameter count and active parameter count are different:

- All expert weights may still need to reside in accelerator memory, or be distributed/offloaded.
- Expert-parallel deployments must send tokens to the devices holding the selected experts.
- Uneven routing can overload some experts while others sit idle; dispatch and all-to-all communication can become bottlenecks.

MoE is an architecture and routing-inside-the-model choice, not a serving request router. For a useful deployment, measure total resident weight memory, active computation, routing balance, and interconnect traffic separately.

## LLM routing: choose a model per request

An **LLM request router** selects among models or providers using request features and service objectives such as quality, cost, latency, context limits, availability, and data policy. A simple strategy sends easy requests to a smaller model and escalates difficult ones to a stronger model. Whether a request is "easy" must be validated against task outcomes, not guessed from prompt length alone.

Common strategies:

1. **Rule-based:** use explicit features such as task type, required context window, modality, or policy. It is transparent and cheap, but rules miss nuance and can become brittle.
2. **Classifier-based:** train a lightweight classifier from labeled requests or outcomes to predict a suitable model/tier. It adds a training and monitoring requirement.
3. **Embedding/similarity-based:** compare the request with examples or task clusters that have performed well on particular models. It is flexible, but depends on representative examples and a sensible similarity metric.
4. **LLM-as-router:** ask a model to categorize the request or select a tool/model. It can handle nuanced instructions, but the routing call adds latency and cost and may itself be unreliable.
5. **Cascade:** try a cheaper/smaller option first, then escalate when a calibrated evaluator, verifier, or task-specific condition indicates that the result is insufficient. The evaluator's errors and the extra round-trip are part of the cost.

The routing objective can be stated as minimizing expected cost/latency subject to a quality and policy constraint:

```text
choose model m for request x
to minimize expected serving cost and latency,
subject to quality(x, m) ≥ target and policy(x, m) is allowed
```

In practice, quality is not known before answering. Build evaluation sets from representative traffic, measure quality and latency per tier, calibrate escalation decisions, and track route distribution, failure rates, tail latency, and spend. Include fallbacks for model outages and avoid using uncalibrated self-reported confidence as the only escalation signal.

## Three different kinds of "router"

The word **router** appears at multiple levels:

| Router | Decision | Timescale |
| --- | --- | --- |
| LLM request router | Which model/provider handles a request | Once or a few times per request |
| MoE router | Which expert(s) process a token in a layer | Many times within one request |
| Serving/disaggregation router | Which prefill/decode worker receives work | At request/phase scheduling time |

They solve different problems and may coexist. In particular, model routing does not itself split a model's prefill and decode phases.

## When to use which approach

| Need | First option to investigate | Main validation |
| --- | --- | --- |
| Lower per-request cost for a predictable task | A task-appropriate SLM | Quality and tail latency on the actual task set |
| Make a small model behave more like a capable teacher | Distillation | Student quality, coverage, safety, and inference cost |
| Large total model capacity with sparse per-token compute | MoE architecture | Expert balance, active compute, memory placement, and communication |
| Different requests need different cost/quality tiers | Request routing or a cascade | Quality under each route, route errors, escalation cost, and latency |
| Keep some requests local/offline | On-device model deployment | Memory, thermal/power, update policy, and model quality |

## References

- Amit Shekhar, [Small Language Models (SLMs)](https://outcomeschool.com/blog/small-language-models-slms), [LLM Routing](https://outcomeschool.com/blog/llm-routing), [How does Knowledge Distillation work?](https://outcomeschool.com/blog/how-does-knowledge-distillation-work), and [Mixture of Experts Explained](https://outcomeschool.com/blog/mixture-of-experts).
- Amit Shekhar, [Cloud vs On-device Model Deployment](https://outcomeschool.com/blog/cloud-vs-on-device-model-deployment).
- Geoffrey Hinton, Oriol Vinyals, and Jeff Dean, [Distilling the Knowledge in a Neural Network](https://arxiv.org/abs/1503.02531).

Sources accessed 2026-10-30.
