# AI/ML system design interviews

[HLD/LLD interview hub](README.md) · [HLD interview guide](hld-interview-guide.md) · [Inference engineering map](../../08-inference-engineering/MAP.md)

## First identify the interview type

“ML system design” is not one standardized interview. Ask the recruiter what role and format the interview targets. Common categories:

| Interview type | What it evaluates | Example question |
| --- | --- | --- |
| Applied ML system design | Turning a business goal into a data/model/serving/monitoring solution | Design recommendations or fraud detection |
| ML infrastructure design | Scalable training, feature, model-serving, orchestration, and resource infrastructure | Design a feature store or distributed training system |
| AI/GenAI application design | Retrieval, prompting, model/provider orchestration, tools, safety, and product evaluation | Design a RAG assistant or LLM gateway |
| Research / research engineering | Research depth, algorithm design, reproduction, experimentation, or performance work | Reproduce and optimize a model/paper |

These tracks overlap but are not interchangeable. An applied-ML guide may not prepare someone for GPU infrastructure or research interviews. Calibrate preparation to the role instead of beginning with a topic list.

## A reusable interview framework

1. **Frame the product problem.** Who is the user, what decision/action should improve, and what is the non-ML baseline? Is ML actually justified?
2. **Define success and constraints.** Business metric, model metric, latency, availability, privacy, cost, and human-review requirements.
3. **Define data and labels.** Sources, permissions, quality, freshness, coverage, label generation, leakage risk, bias, and feedback loops.
4. **Choose a formulation.** Classification, ranking, retrieval, generation, forecasting, or rules; explain why it maps to the objective.
5. **Design the lifecycle.** Ingestion → transformation/features → training/evaluation → versioned deployment → inference → monitoring/retraining.
6. **Design online serving.** Request path, model/version selection, feature availability, batching/cache, latency budget, scaling, fallback, and rollout.
7. **Evaluate and operate.** Offline/online metrics, experiments, drift/quality alerts, rollback, incident playbooks, and human feedback.
8. **Deep dive and summarize.** Select the largest risk; compare one alternative; state limitations and what you would measure next.

Keep a separation between:

- **Business metric:** the outcome the product cares about.
- **Model metric:** an offline measure such as precision/recall, NDCG, calibration, or task quality.
- **System metric:** latency, throughput, availability, GPU/CPU/memory, queue depth, and cost.

Optimizing one does not guarantee improvement in the others. A higher offline score can harm latency, fairness, or user trust.

## Applied ML design essentials

### Problem and data

- Establish whether a rules-based or retrieval baseline is adequate before proposing a model.
- Define the prediction unit, label, prediction time, and when the outcome becomes observable.
- Prevent **target leakage**: training features must be available at the time the production prediction is made.
- Separate training and evaluation data by time/user/entity where random splitting would leak correlated examples.
- Address missing data, class imbalance, label delay/noise, privacy, consent, and retention.
- Make feedback loops explicit: recommendations change what users see, which changes future training data.

### Training and deployment

- Version code, data snapshot, feature definitions, model artifact, and evaluation results.
- Use reproducible training and data validation; track lineage and rollback candidates.
- Choose batch, streaming, or online feature computation from freshness and latency requirements.
- Use canary/shadow/A-B rollout where appropriate; define stop/rollback metrics.
- Keep model serving separate from noncritical training/analytics jobs if resource contention matters.

### Monitoring

Monitor not just infrastructure health but input quality, feature freshness, prediction distributions, delayed-label model quality, segment-level performance, and business outcomes. Data/model drift is a signal to investigate, not proof that model quality has fallen. Define human escalation and safe fallback behavior.

## GenAI and retrieval-augmented generation

A common RAG request path:

```text
Sources -> parse/clean -> chunk + metadata -> embed -> vector/search index
                                                         |
User query -> auth/tenant filter -> retrieve -> rerank -> prompt/context
                                                  -> model inference -> validate/output
```

Discuss:

- Ingestion freshness, document permissions, deletion, duplicate detection, chunk boundaries, and metadata.
- Embedding model/version compatibility and re-index/re-embed strategy.
- Retrieval quality (recall/precision), hybrid keyword/vector search, filters, and reranking.
- Prompt/context size, citation/evidence requirements, hallucination risk, and output validation.
- Per-tenant isolation and access checks at retrieval time; retrieved content is untrusted input.
- Streaming vs full response, timeouts, retries, caching, quotas, fallback models, and per-request cost.
- Evaluation datasets and separate measurements for retrieval, answer relevance, groundedness, refusal, latency, and safety.
- Tool calls as privileged actions: schema validation, authorization, sandboxing/allowlists, audit, and idempotency.

RAG is not automatically the right answer. If the data is small/stable, context can be directly supplied; if the task is structured and deterministic, a normal query/API may be more accurate and cheaper.

## LLM gateway and inference-serving design

### Multi-provider gateway

Separate a stable application contract from provider-specific adapters. A routing policy can select a model by capability, latency, availability, data-residency constraint, or budget. Define:

- Provider/model capability normalization and unsupported features.
- Credential isolation, tenant quotas, rate limits, and cost attribution.
- Request/response schema, streaming cancellation, retry policy, and provider error mapping.
- Circuit breaking, fallback semantics, idempotency, and whether fallback changes output quality.
- Cache key/privacy, retention, prompt-injection/data leakage risk, and audit requirements.
- Evaluation before routing a new provider/model into production.

Do not claim provider interchangeability unless semantic differences, tool behavior, safety, and response formats are handled.

### Model-serving platform

For infrastructure-oriented roles, reason about request scheduling and hardware:

- Model loading, memory capacity, tensor/pipeline/data parallelism as required by model/hardware.
- Prefill vs decode compute, batching, continuous scheduling, and fairness across requests.
- KV-cache memory use, admission control, context-length limits, and preemption.
- Quantization/speculative decoding trade-offs in quality, latency, throughput, and hardware support.
- Streaming, time-to-first-token, inter-token latency, request completion latency, throughput, and tail behavior.
- Autoscaling lag, warm pools, model download/startup time, GPU fragmentation, and graceful overload.
- Isolation across tenants, payload logging policy, and quotas/cost controls.

Use the [inference-engineering notes](../../08-inference-engineering/MAP.md) for deeper technical details; do not turn a product-level interview into a kernel-level discussion unless the role requires it.

## AI/ML interview practice questions

| Prompt | Main design dimensions |
| --- | --- |
| Design an e-commerce recommendation system | Candidate generation, ranking, freshness, feedback loops, online/offline features |
| Design fraud detection | Label delay, class imbalance, latency, precision/recall cost, human review |
| Design content moderation | Policy/versioning, multimodal inputs, safety, thresholds, appeals, audit |
| Design a RAG platform | Ingestion, permissions, retrieval, reranking, evaluation, deletion |
| Design an LLM gateway | Provider adapters, routing, quotas, fallbacks, observability, cost |
| Design an AI agent platform | Tool registry, permissions, state, retries, timeouts, step limits, audit |
| Design an LLM evaluation platform | Datasets, graders, reproducibility, human review, regression gates |
| Design a code-generation service | Repo indexing, context selection, sandbox, privacy, test feedback |
| Design inference serving | Scheduler, batching, KV cache, hardware utilization, SLOs |
| Design a multi-tenant AI platform | Tenant isolation, access control, resource quotas, data retention, cost allocation |

For each prompt, write down one product metric, one model/quality metric, one system SLO, the data flow, and one failure/abuse mode before drawing components.

## Sources and further reading

Inspected on **2026-11-06**:

- [Hello Interview: ML System Design in a Hurry](https://www.hellointerview.com/learn/ml-system-design/in-a-hurry/introduction) differentiates applied ML, ML infrastructure, research, and research engineering interviews. Its introduction discusses the applied-ML framing, data/features/labels, model design, and interview assessment. Content may evolve and some lessons may be gated.
- [Evidently documentation](https://docs.evidentlyai.com/introduction.md) links to [LLM evaluation](https://docs.evidentlyai.com/quickstart_llm.md), [ML monitoring](https://docs.evidentlyai.com/quickstart_ml.md), a metric catalog, and examples. It is a product/library reference, not a neutral evaluation standard.
- [vLLM documentation](https://docs.vllm.ai/en/stable/) links to the [quickstart](https://docs.vllm.ai/en/stable/getting_started/quickstart/), [user guide](https://docs.vllm.ai/en/stable/usage/), and [developer guide](https://docs.vllm.ai/en/stable/contributing/). Its overview describes PagedAttention, continuous batching, KV-cache management, streaming, quantization, parallelism, and supported model families; exact capabilities change by release and hardware.
- [ByteByteGo System Design 101](https://github.com/ByteByteGoHq/system-design-101) includes an [AI/ML section](https://bytebytego.com/guides/ai-machine-learning) and case-study links such as [how ChatGPT works](https://bytebytego.com/guides/how-does-chatgpt-work). Use it as a visual overview and verify details against current serving/model documentation.
- [System Design Primer](https://github.com/donnemartin/system-design-primer) supplies broad architecture practice and interview examples.
- The original roadmap named *Machine Learning System Design Interview* by Ali Aminian and Alex Xu but provided no destination URL. No publisher link is invented here.

