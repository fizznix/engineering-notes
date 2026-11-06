# Observability and telemetry

[Back to Cloud and DevOps](README.md) · [System Design](../04-system-design/README.md) · [Messaging and event streaming](../09-data-engineering/messaging-and-event-streaming.md)

Observability is the ability to understand a system's internal behavior from the information it emits, including when a failure was not anticipated in advance. Telemetry—metrics, logs, traces, and related signals—is the evidence. Monitoring, dashboards, alerts, and incident response are practices built on that evidence.

The goal is not to collect every possible field. It is to answer useful questions about user-visible health, find the cause of failures, and make safe operational decisions without leaking sensitive data or overwhelming people and systems with noise.

## Observability, monitoring, and telemetry

- **Telemetry:** data emitted about a running system and its behavior.
- **Instrumentation:** code/library/agent that records and exports telemetry.
- **Monitoring:** continuously checking selected measurements and behavior against expectations.
- **Observability:** the ability to ask useful questions about internal state from emitted evidence, including novel failure modes.
- **Dashboard:** a curated view for understanding a service or investigating a symptom; not a substitute for alerting or diagnosis.
- **Alert:** notification that a person or automation should act; a page should indicate an urgent, actionable user/service problem.

You cannot make a system observable only by purchasing a backend. The application must emit the right signals at useful boundaries, propagate context across services, and retain enough data for the required questions.

## The signals and when each helps

| Signal | Represents | Best for | Typical dimensions/fields |
| --- | --- | --- | --- |
| **Metrics** | Numeric measurements aggregated over time | Rates, errors, latency distributions, saturation, capacity, SLOs/alerts | Service, route template, region, status class |
| **Logs** | Timestamped records of individual events (structured where possible) | What happened to a particular request/entity and the detail around an error | Timestamp, severity, message, trace/span ID, safe domain attributes |
| **Traces** | A causal path composed of spans across one request/workflow | Where time was spent, which dependency failed, fan-out/critical path | Trace ID, parent span, service, operation, status, duration |
| **Profiles** | Code/resource usage over time | CPU/memory hot paths and allocation behavior | Process, function/stack, resource |

Use them together:

- Metrics tell you **how often/how much** and whether a population is unhealthy.
- Traces show **where one sampled request spent time** across boundaries.
- Logs provide **event-level context** for a particular operation.
- Profiles help explain **which code path consumes resources**.

No signal replaces the others. A trace can be sampled and absent for a request; a metric can show a service-wide failure rate without explaining a specific request; a log can explain one request but not reliably quantify the service-wide error budget.

## OpenTelemetry: instrumentation and collection

OpenTelemetry (OTel) is a vendor-neutral framework for instrumentation, generation, collection, and export of telemetry. It is not itself a metrics database, log-search backend, tracing UI, or complete alerting system.

### Main building blocks

- **API:** instrumentation calls/types used by application or libraries.
- **SDK:** implementation that records, aggregates/samples, and exports telemetry.
- **Instrumentation libraries/agents:** capture application/framework/runtime behavior automatically or through code.
- **Resource attributes:** identify the service/process/environment, such as service name, version, and deployment environment.
- **Semantic conventions:** shared attribute names for common operations and resources; consistent names make cross-service queries possible.
- **Context propagation:** carries trace context across process/network boundaries (commonly W3C Trace Context) so downstream spans join the same causal trace.
- **Collector:** vendor-neutral pipeline with receivers, processors, and exporters for receiving, filtering/enriching/batching, and sending telemetry onward.
- **Backend:** stores/indexes/queries data, presents dashboards/traces, and/or evaluates alerts; choose separately.

Typical deployment:

```text
Application (OTel API + SDK)
  -> OTLP export
  -> OpenTelemetry Collector
       receivers -> processors (batch, redact/filter, enrich) -> exporters
  -> metrics backend / trace backend / log backend
  -> dashboards and actionable alerting
```

For development or a small deployment, direct export may be adequate. A Collector can centralize retries, batching, filtering/redaction, routing, and backend changes. It also becomes an operational component: monitor its queue/memory/export failures, size it, and prevent it from becoming a silent telemetry-loss point.

### Traces and context

A **trace** represents one operation across services. Each **span** records a timed unit of work, its parent/child relationship, status, and attributes/events. Propagate trace context through HTTP headers, messaging metadata, and asynchronous jobs. A message should preserve the producer trace context so the consumer can continue or link a trace according to the framework's messaging conventions.

Use semantic attributes such as service/route/operation rather than embedding request bodies. Keep trace IDs in logs to pivot between signals. Baggage propagates contextual key/value data; do not put secrets, PII, or unbounded values into it.

Sampling controls trace volume. Head sampling decides early and is economical; tail sampling can use completed trace characteristics but needs buffering/state and careful collector capacity. Ensure error/high-latency traces are not accidentally excluded by a policy meant only to reduce normal traffic volume.

### Logs

Prefer structured logs with stable fields, severity, timestamp, service/version, and correlation IDs. Log once at meaningful boundaries; avoid logging the same exception at every layer. Keep sensitive data and high-volume debug payloads out of production logs by default. A logging bridge/collector can normalize existing logs into an OTel-compatible pipeline.

## Prometheus: metrics collection and alerting

Prometheus is an open-source metrics monitoring/alerting toolkit. Its core model stores timestamped time series identified by a metric name and label set. A Prometheus server commonly scrapes targets over HTTP; Pushgateway is intended for short-lived batch jobs that cannot be scraped directly, not a general replacement for pull-based service instrumentation.

### Metric types

- **Counter:** cumulative value that increases (or resets on process restart), such as requests or completed jobs. Query rates over a window rather than treating raw counter values as a current level.
- **Gauge:** value that can rise or fall, such as in-flight requests, queue depth, or memory in use.
- **Histogram:** records observations into buckets plus count/sum; bucket data can be aggregated across instances to estimate distributions/quantiles. Choose boundaries that resolve the SLO thresholds you care about.
- **Summary:** often exports client-calculated quantiles plus count/sum. Precomputed quantiles generally cannot be meaningfully aggregated across replicas in the way histogram buckets can.

Prometheus also supports native histograms; exact availability and query behavior vary with Prometheus and client versions. Prefer a distribution representation that can answer the actual aggregate questions, and consult current histogram guidance.

### Labels and cardinality

Each unique metric name/label-value combination creates a time series. Labels enable dimensional aggregation but multiply series. Do not use user ID, request ID, order ID, raw URL, or other unbounded identifiers as Prometheus labels. Prefer bounded labels such as route templates (`/orders/{id}`), method, result class, region, and service. Put per-request identifiers in logs/traces where they can be sampled/indexed under appropriate controls.

Keep metric names/units stable (`_total` for counters by convention, seconds for duration). Avoid labels that encode values as metric names or create one series per event.

### Queries and alerts

Example request rate:

```promql
sum by (service, route) (
  rate(http_server_requests_total[5m])
)
```

Example fraction of successful requests:

```promql
sum(rate(http_server_requests_total{outcome="success"}[5m]))
/
sum(rate(http_server_requests_total[5m]))
```

In production, define which requests count as eligible/good, handle absent/low traffic appropriately, and avoid treating every `4xx` or expected business rejection as a server failure.

Prometheus alerting is split:

1. Prometheus evaluates alerting rules over PromQL expressions.
2. Alertmanager groups, routes, silences, inhibits, and sends notifications.

Use `for` durations to avoid paging on a one-scrape blip when appropriate; ensure this does not delay a critical alert beyond the error budget. Labels route ownership/severity; annotations explain user impact and next steps. Test alert rules and routing.

Prometheus is not a general event log or high-cardinality analytics store. Its overview describes autonomous server nodes rather than a distributed storage layer; plan retention, remote storage/federation, and high availability based on the deployment requirements.

## SRE: measure reliability from the user's perspective

### SLI, SLO, SLA, and error budget

- **SLI (Service Level Indicator):** a defined quantitative measurement of a service behavior that matters to users (for example, successful requests or latency under a threshold).
- **SLO (Service Level Objective):** target for an SLI over a window (for example, 99.9% of eligible requests succeed over 30 days).
- **SLA (Service Level Agreement):** an external/business commitment and consequences; not just another name for an internal SLO.
- **Error budget:** the allowed fraction/count of bad events under the SLO. For a 99.9% request-success objective, the budget is 0.1% of eligible events in the window. An exhausted/rapidly burning budget should change release/mitigation priorities.

Specify numerator, denominator, window, exclusions, and data source. Measure as close to the user's observed behavior as practical; server uptime alone is not a user-facing availability SLI.

### Golden signals and complementary views

The SRE book's “four golden signals” frame service monitoring:

- **Latency:** successful and failed request durations should be distinguished.
- **Traffic:** demand through the service, not just machine utilization.
- **Errors:** explicit failures and unexpected/incorrect results.
- **Saturation:** how close constrained resources are to their limits (queueing, worker pools, CPU, memory, connections).

Complement with:

- **RED** for request-driven services: Rate, Errors, Duration.
- **USE** for resources: Utilization, Saturation, Errors.

These are coverage heuristics, not an SLO by themselves. A graph for every metric does not ensure the service's key behavior is measured.

### Alerting and incident response

- Page for a current, user-impacting or imminent reliability issue that requires timely human action.
- Use tickets/reports for non-urgent capacity, maintenance, or trend work.
- Prefer SLO/error-budget burn and symptoms over paging on every low-level dependency fluctuation.
- Use multi-window burn-rate logic to catch both fast severe incidents and slower sustained budget loss.
- Avoid noisy thresholds that flap; group related alerts and document owner, impact, dashboard, and immediate mitigation.
- Treat runbooks as decision support; automate safe repetitive mitigation where practical.
- Review incidents and near misses to improve instrumentation, SLOs, architecture, and runbooks.

Monitoring helps understand a problem; an alert is a signal to act. A dashboard should not require someone to stare at it continuously to detect an outage.

## Instrumentation by workload

### Online request service

Record request count, result/outcome, duration distribution, and in-flight work at the service boundary. Use route templates rather than raw paths. Add dependency spans/metrics for database, queue, and external calls. Compare client-visible latency with server-side processing latency.

### Queue/stream consumer

Monitor:

- Publish/consume/ack/failure rates.
- Queue depth and oldest message age.
- Kafka consumer lag and partition skew; RabbitMQ ready/unacknowledged message counts and consumer capacity.
- Redeliveries, retries, dead-letter count/age, rebalance/assignment changes, and processing duration.
- Downstream dependency saturation and message age at user-visible completion.

For asynchronous products, “request accepted” availability and “work completed within X” freshness are distinct SLIs.

### Database

Monitor query latency, slow-query distribution, error/timeout rate, active/idle/pool-wait connections, lock waits/deadlocks, CPU/I/O/storage, replication lag, backup age/restore tests, and maintenance health. Avoid exposing SQL parameter values or user identifiers as metric labels.

## Worked path: diagnose a slow checkout

1. **SLO alert:** checkout latency/error budget indicates user impact.
2. **Metrics:** request histogram shows p99 regression; traffic is stable; saturation shows connection-pool wait increasing.
3. **Trace:** slow samples show time concentrated in an inventory database span rather than payment/provider calls.
4. **Logs:** correlated error records show lock wait/deadlock details with safe query/transaction metadata, not raw card data.
5. **Action:** mitigate pool/traffic/transaction issue, verify latency and error-budget burn recover, then investigate the query/lock pattern.

Without trace context, one may inspect unrelated logs. Without metrics, one slow trace may be anecdotal. Without user-facing SLOs, the team may optimize a database metric while checkout remains broken.

## Design and operational checklist

- Does each service have user-relevant SLIs and reviewable SLOs?
- Are latency, traffic, errors, and saturation covered for online services?
- Are queue backlog age, consumer lag, and async completion freshness measured?
- Are trace context and correlation IDs propagated across HTTP and messaging boundaries?
- Are metric labels bounded and semantically stable?
- Can logs/traces link to the same request without exposing secrets/PII?
- Are sampling, retention, access control, and telemetry export costs understood?
- Is collector/exporter failure visible, and can it avoid blocking application requests?
- Do alerts have an owner, actionable condition, and useful runbook?
- Are dashboards supplemental diagnostics rather than the only outage detector?
- Are alert/SLI rules tested against normal, low-traffic, and failure scenarios?

## Further reading

Inspected on **2026-11-06**:

- [OpenTelemetry documentation](https://opentelemetry.io/docs/) links to [getting started](https://opentelemetry.io/docs/getting-started/), [concepts](https://opentelemetry.io/docs/concepts/), [observability primer](https://opentelemetry.io/docs/concepts/observability-primer/), [signals](https://opentelemetry.io/docs/concepts/signals/), [traces](https://opentelemetry.io/docs/concepts/signals/traces/), [metrics](https://opentelemetry.io/docs/concepts/signals/metrics/), [logs](https://opentelemetry.io/docs/concepts/signals/logs/), [context propagation](https://opentelemetry.io/docs/concepts/context-propagation/), [semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/), and the [Collector](https://opentelemetry.io/docs/collector/). The Collector page explains when direct export is sufficient and why a Collector helps with batching, retries, encryption, or sensitive-data filtering.
- [Google SRE book contents](https://sre.google/sre-book/table-of-contents/) links to [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/), [Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/), [Practical Alerting](https://sre.google/sre-book/practical-alerting/), [Handling Overload](https://sre.google/sre-book/handling-overload/), [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/), and [Data Integrity](https://sre.google/sre-book/data-integrity/). The [SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) develops SLO-based alerting and burn-rate decisions.
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/) links to its [data model](https://prometheus.io/docs/concepts/data_model/), [metric types](https://prometheus.io/docs/concepts/metric_types/), [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/), [histograms and summaries](https://prometheus.io/docs/practices/histograms/), [instrumentation practices](https://prometheus.io/docs/practices/instrumentation/), [alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/), [Alertmanager overview](https://prometheus.io/docs/alerting/latest/overview/), and [Pushgateway guidance](https://prometheus.io/docs/instrumenting/pushing/). The [getting-started guide](https://prometheus.io/docs/prometheus/latest/getting_started/) demonstrates scraping, queries, rules, and graphs.
