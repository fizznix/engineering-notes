# HLD and LLD interview preparation

[Back to Interview Preparation](../README.md) · [System Design](../../04-system-design/README.md) · [Repository index](../../README.md)

This package teaches two related but different interview skills:

- **High-Level Design (HLD):** turn a product or infrastructure need into a system architecture; explain component boundaries, data flow, scaling, consistency, reliability, security, and cost.
- **Low-Level Design (LLD), also called Object-Oriented Design (OOD):** turn a bounded feature into responsibilities, state, contracts, collaborations, and working code or pseudocode.

Neither interview has one canonical solution. The interviewer is evaluating how you frame the problem, make decisions, adapt to constraints, and communicate—not whether you can reproduce a memorized diagram.

## Guide map

| Guide | Use it to learn |
| --- | --- |
| [HLD interview guide](hld-interview-guide.md) | A delivery framework, system-design fundamentals, estimation, trade-offs, a worked URL-shortener example, and practice questions. |
| [LLD interview guide](lld-interview-guide.md) | A delivery framework, object modeling, UML essentials, a worked parking-lot example, machine-coding practice, and common problem types. |
| [AI/ML system design interviews](ai-ml-system-design-interviews.md) | Applied ML and GenAI problem framing, data/model/serving/evaluation flows, and specialty interview questions. |
| [Software design and code quality](../../02-programming/software-design/README.md) | The longer reference for clean code, SOLID, GoF patterns, refactoring, and architecture patterns. |
| [System design and architecture notes](../../04-system-design/README.md) | Related system-design concepts, including an evolutionary scaling walkthrough. |

## HLD and LLD work together

| Question | HLD | LLD |
| --- | --- | --- |
| Main scale | Product, service, and distributed-system boundaries | A feature, module, or bounded application component |
| Typical artifacts | Requirements, APIs, data model, component/data-flow diagram, failure and scaling choices | Use cases, entities/value objects, contracts, object relationships, key method logic, tests |
| Main risks | Wrong requirements, unexamined bottlenecks, consistency/failure gaps, overbuilt architecture | Unclear ownership, invalid state, coupling, concurrency bugs, over-abstraction |
| Common interview mode | Discussion and a shared architecture whiteboard | Object design discussion, pseudocode, or machine-coding exercise |
| What the interviewer wants | A system that meets requirements and can evolve under realistic constraints | A coherent, testable design that implements the important behavior correctly |

The boundary depends on the prompt. An HLD answer may briefly specify an interface or data invariant; an LLD answer may mention the service boundary where the component will run. Do not spend most of an HLD interview drawing classes, or most of an LLD interview inventing a globally distributed deployment.

## A flexible 10-week preparation path

This is a menu, not a guarantee or a fixed prerequisite chain. If time is short, prioritize the skills that are weakest and do at least one timed practice instead of reading every linked resource. A reasonable practice block is 60–120 focused minutes; consistency and active recall matter more than a prescribed daily duration.

| Week | Focus | Evidence of progress |
| --- | --- | --- |
| 1 | HLD framing, networking, HTTP/API basics | Explain a request path and distinguish a proxy, load balancer, gateway, and CDN. |
| 2 | Data modeling, SQL/NoSQL, indexes, transactions, consistency | Choose storage from access patterns and invariants; explain the cost of an index or replica. |
| 3 | Caching, queues, event delivery, idempotency | Draw a read/write path and explain staleness, duplicates, retries, and backpressure. |
| 4 | Reliability, observability, security, deployment | State failure behavior, SLO/SLI, recovery expectations, and trust boundaries. |
| 5 | HLD case studies | Complete a URL shortener, rate limiter, notification system, and chat design under time. |
| 6 | LLD and OOP fundamentals | Turn use cases into a small model with clear ownership, contracts, and valid states. |
| 7 | SOLID, patterns, clean code, tests | Add an abstraction only where variation or a boundary warrants it; explain the cost. |
| 8 | LLD and machine coding | Implement core paths for parking lot, vending machine, Splitwise, or an elevator and test edge cases. |
| 9 | Specialty design | Practice ML/GenAI or another role-specific track after core HLD and LLD are stable. |
| 10 | Mocks and targeted revision | Complete timed HLD/LLD mocks, review feedback, and re-solve weak areas from a blank page. |

### Weekly practice loop

1. **Retrieve:** write down what you remember about one concept before rereading it.
2. **Apply:** solve one new problem with a timer and a blank diagram/editor.
3. **Explain:** speak through assumptions, alternatives, and failure cases as if the interviewer were present.
4. **Review:** compare against a source only after attempting the problem; note missed requirements, wrong assumptions, weak trade-offs, and time loss.
5. **Repeat:** revisit the same problem after several days, then solve a related variant.

Keep a small mistake log with categories such as requirements, data model, scaling, consistency, failure handling, object ownership, implementation, testing, and communication. “I need to study more” is not a useful diagnosis; “I did not make spot allocation atomic under concurrency” is.

## How to run a mock

- Ask the partner to give only the prompt and answer clarifying questions as an interviewer would.
- Use a shared whiteboard or editor and the same time limit as the target interview.
- Ask for feedback on problem framing, prioritization, technical reasoning, correctness, and communication separately.
- Repeat the same question only after writing down the feedback; then try a new question to test transfer.
- Ask the recruiter about interview format, allowed tools, language, seniority expectations, and whether LLD means discussion, pseudocode, or executable code.

### Readiness signals

You are approaching readiness when you can repeatedly:

- Clarify a vague prompt and agree on a bounded scope before designing.
- Deliver a complete baseline before diving deeply into optional components.
- Connect each component, table, queue, class, or interface to a requirement.
- State assumptions, quantify only the quantities that matter, and revise when assumptions change.
- Explain the primary alternative and what it would cost.
- Handle a new interviewer constraint without defending an obsolete design.
- Trace a concrete request through the system or a concrete scenario through the object model.
- Surface failure paths, tests, and security considerations without losing the main thread.

Passing cannot be guaranteed by finishing a course or a problem count. Role requirements and interview rubrics vary; prioritize recruiter/interviewer guidance and practice under realistic conditions.

## Reading the source material

The original [HLD/LLD roadmap](../../design-pattern.md) contained resource names but no actual hyperlink destinations. These guides resolve the names to canonical references, summarize relevant subtopics in original language, and call out source pages that were inaccessible or gated. External material is optional further reading; the core learning material is captured in this package.

### HLD resources

- [Hello Interview: System Design in a Hurry](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction) links to a [delivery framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery), [core concepts](https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts), [key technologies](https://www.hellointerview.com/learn/system-design/in-a-hurry/key-technologies), and [how to prepare](https://www.hellointerview.com/learn/system-design/in-a-hurry/how-to-prepare). It distinguishes problem navigation, solution design, technical depth, and collaboration. Some problem breakdowns and guided practice are premium; this note does not depend on gated content.
- [System Design Primer](https://github.com/donnemartin/system-design-primer) is an open, broad topic index with an [interview approach](https://github.com/donnemartin/system-design-primer#how-to-approach-a-system-design-interview-question), sample [system-design solutions](https://github.com/donnemartin/system-design-primer#system-design-interview-questions-with-solutions), and [object-oriented-design solutions](https://github.com/donnemartin/system-design-primer#object-oriented-design-interview-questions-with-solutions). Its topic sublinks cover scalability, performance/latency, consistency and CAP, DNS/CDNs, load balancing, databases, caching, asynchronous queues, communication, and security.
- [ByteByteGo's System Design 101 repository](https://github.com/ByteByteGoHq/system-design-101) organizes illustrated reading into [API and web development](https://github.com/ByteByteGoHq/system-design-101#table-of-contents), [real-world case studies](https://bytebytego.com/guides/real-world-case-studies), [database and storage](https://bytebytego.com/guides/database-and-storage), and AI/ML. Its companion index links focused articles on HTTP, load balancers, APIs, queues, delivery semantics, data stores, and production architectures. Treat illustrations as a starting model; validate the underlying trade-offs.
- [MDN HTTP documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP) links to [HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview), [protocol evolution](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP), [caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching), [authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Authentication), [proxies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Proxy_servers_and_tunneling), and WebSocket upgrade guidance. Use it for protocol semantics, not as a system-design interview script.
- [Cloudflare Learning Center](https://www.cloudflare.com/learning/) provides primers on DNS, CDNs, TLS, networking, cloud, and streaming. It is an accessible overview, not a replacement for protocol specifications or vendor-neutral architecture reasoning.
- [Designing Data-Intensive Applications](https://dataintensive.net/) is an advanced book reference for storage/processing trade-offs, replication, partitioning, transactions, and consistency.

### LLD resources

- [Hello Interview: Low-Level Design in a Hurry](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/introduction) links to its [delivery framework](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/delivery), [design principles](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/design-principles), [OOP concepts](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/oop-concepts), [patterns](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/patterns), and problem breakdowns such as [parking lot](https://www.hellointerview.com/learn/low-level-design/problem-breakdowns/parking-lot). The parking-lot article exposes requirements and indicates later material is premium; this local guide contains its own complete original example.
- [Awesome Low Level Design](https://github.com/ashishps1/awesome-low-level-design) groups learning links for [OOP](https://github.com/ashishps1/awesome-low-level-design#-oop-fundamentals), [class relationships](https://github.com/ashishps1/awesome-low-level-design#-class-relationships), [design principles](https://github.com/ashishps1/awesome-low-level-design#-design-principles), [patterns](https://github.com/ashishps1/awesome-low-level-design#-design-patterns), [UML](https://github.com/ashishps1/awesome-low-level-design#-uml), concurrency, and interview questions.
- [Java Design Patterns](https://github.com/iluwatar/java-design-patterns) links to a [principles catalog](https://java-design-patterns.com/principles/) and [pattern catalog](https://java-design-patterns.com/patterns/) with commented Java implementations. Its guidance starts with simplicity and adding complexity only when needed.
- [UML Diagrams](https://www.uml-diagrams.org/) explains UML notation and its scope. UML is a modeling language, not a development process; use lightweight boxes/arrows in interviews unless formal notation is requested.
- The [local software-design notes](../../02-programming/software-design/README.md) contain the detailed pattern, SOLID, clean-code, and refactoring references relevant to LLD.

### AI/ML system design resources

- [Hello Interview: ML System Design in a Hurry](https://www.hellointerview.com/learn/ml-system-design/in-a-hurry/introduction) distinguishes applied ML system design, ML infrastructure, research, and research engineering interviews. Its accessible introduction discusses problem framing, data/features/labels, model design, and the interview rubric; page depth and access may change.
- [Evidently documentation](https://docs.evidentlyai.com/introduction.md) links to [LLM evaluation](https://docs.evidentlyai.com/quickstart_llm.md), [ML monitoring](https://docs.evidentlyai.com/quickstart_ml.md), metrics, and examples.
- [vLLM documentation](https://docs.vllm.ai/en/stable/) links to quickstart, user, and developer guides and describes serving concerns such as batching, KV-cache management, streaming, quantization, and parallelism.
- The [local inference-engineering map](../../08-inference-engineering/MAP.md) is the deeper reference for serving internals. The source outline also named *Machine Learning System Design Interview* by Ali Aminian and Alex Xu but supplied no URL; no unverified publisher URL is invented here.

All web sources were checked on **2026-11-06**. The original source labels included PostgreSQL, Redis, Kafka, RabbitMQ, OpenTelemetry, Prometheus, and Google's SRE book. Their focused primary documentation is linked from the relevant HLD and AI/ML guides; the explanations remain available locally.

