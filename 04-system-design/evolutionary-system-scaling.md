# System Design & Architecture: The Evolutionary Approach

These learning notes follow the progression presented in the source video, from a single-server baseline through scaling compute, state, and data. Examples and numeric values are illustrative; real system behavior depends on workload and implementation.

## 0. Baseline Architecture: The Single Server Setup

* **Components:** 1 Application Server (e.g., Node.js/Express, Django, or Go) + 1 Relational Database (e.g., PostgreSQL or MySQL) co-located or on adjacent VMs.
* **Context:** This setup is capable of handling thousands of users (not just dozens) and up to millions of requests per day if optimized with basic connection pooling and indexes. A vast majority of applications, internal tools, and early-stage startups run on this setup and never need to grow beyond it.
* **How & Why it Breaks:**
* **CPU/Memory Starvation:** Compute-bound tasks (e.g., PDF generation, heavy JSON serialization) exhaust OS threads or CPU cycles.
* **Connection Limits:** Incoming TCP connections hit `max_connections` or OS file descriptor limits (`ulimit`).
* **Request Backlog:** Under sudden traffic spikes (e.g., ticket flash sales), requests pile up in the OS/web server listen socket backlog (`SOMAXCONN`). When the queue fills up, new requests fail with TCP resets (`ECONNREFUSED`) or HTTP `504 Gateway Timeout` errors as client timeouts trigger before the server touches the request.



---

## Phase 1: Scaling Compute (Handling Traffic Spikes)

When a single server runs out of processing capacity, you have two choices: **Vertical Scaling** or **Horizontal Scaling**.

### Option A: Vertical Scaling (Scale Up)

* **What it is:** Upgrading the hardware specs of the existing machine (increasing CPU cores, RAM, network bandwidth, or moving from standard SSDs to NVMe drives).
* **Pros:**
* **Zero Code/Architecture Impact:** Keeps the deployment model identical; no network latency added between distributed components.
* **Cost-Efficient Early On:** Upgrading a $20/month instance (2 vCPU, 4GB RAM) to a $160/month instance (16 vCPU, 32GB RAM) is cheaper and faster than re-architecting for distributed computing.


* **Cons/Limits:**
* **Hardware Ceiling:** You eventually hit the maximum physical machine size offered by cloud providers (e.g., AWS `u-24tb1.metal`).
* **Downtime during resized reboots:** Resizing instance types usually requires stopping the instance.
* **Non-linear cost scaling:** High-end enterprise instances grow exponentially in price relative to performance gains.



### Option B: Horizontal Scaling (Scale Out) & Load Balancing

* **What it is:** Running multiple identical instances of your app server across separate servers or containers, splitting incoming traffic across them.
* *Crucial Distinction:* These are duplicate stateless instances running the exact same code, not microservices or different application domains.


* **Routing Strategy:** Requires an edge router or **Load Balancer** (e.g., NGINX, HAProxy, AWS ALB) to act as the reverse proxy.
* **Balancing Algorithms:**
* **Round Robin / Weighted Round Robin:** Even distribution or weighted based on server capacity.
* **Least Connections:** Routes to the instance currently serving the fewest active connections (ideal for long-running HTTP requests).
* **IP Hash:** Routes requests from the same client IP to the same server instance (a naive approach to session sticky routing).




* **Costs & Trade-offs:**
* **Single Point of Failure (SPOF):** If the load balancer crashes, the entire system is inaccessible regardless of how many application instances are running behind it.
* *Mitigation:* Active-Passive or Active-Active Load Balancer pairs using Virtual IPs (VIPs) managed via protocols like Keepalived / VRRP or cloud-native DNS failover (e.g., AWS Route 53 health checks).


* **Interchangeability Requirement:** Applications must become completely stateless. If an application server keeps transient state in process memory, requests routed to alternate servers will fail.



---

## Phase 2: Solving State (Stateless Servers & Shared Session Store)

* **The Problem (Random Logouts):**
* In a traditional monolithic application, sessions are stored in local server RAM (e.g., in-memory hash map or local file system `/tmp/sess_...`).
* If User A logs in and receives a `session_id`, Server 1 records `session_id -> user_id`. On the next request, the load balancer routes User A to Server 2. Server 2 checks its local RAM, finds no record of `session_id`, and returns HTTP `401 Unauthorized`.
* *Result:* With 3 load-balanced servers, roughly ~66% of user interactions result in unexpected logouts.


* **Architectural Fix: Externalizing Session State**
* Application servers must be forced into a **stateless architecture** where local memory holds zero user state between HTTP requests.
* **New Component Introduced:** Centralized In-Memory Data Store (typically **Redis** or **Memcached**).



```
   User Request
        │
        ▼
  Load Balancer (NGINX / ALB)
   ├───► Server A ──┐
   ├───► Server B ──┼──► [ Centralized Redis Session Store ]
   └───► Server C ──┘    (Shared key-value state across all nodes)

```

* **Workflow:**
1. User authenticates against Server A.
2. Server A generates a `session_id` and writes `session_id -> { userId: 42, role: "admin" }` into the central Redis instance (with a set TTL/expiration time).
3. Subsequent requests hit Server B or C. The server reads `session_id` from the incoming HTTP `Cookie` header, executes a sub-millisecond lookup in Redis, and authenticates the user seamlessly.


* **Alternative State Approaches:**
* **Stateless JWTs (JSON Web Tokens):** Cryptographically signed tokens held on the client side containing user claims. Eliminates the session lookup database call entirely, but introduces token revocation challenges (e.g., revoking compromised sessions before expiry requires maintaining a blacklisting queue).


* **Costs & Trade-offs:**
* **Added Latency:** Introduces an extra network round-trip (1–5ms) on every authenticated HTTP request to fetch session details from Redis.
* **Critical Dependency (SPOF):** If Redis goes down, authentication fails system-wide. Requires running Redis in Sentinel or Cluster mode for automatic failover.



---

## Phase 3: Scaling the Database (Connections & Read Heavy Loads)

Once stateless application nodes scale horizontally, the underlying relational database becomes the primary system bottleneck.

### Problem 3.1: Connection Exhaustion & Overhead

* **The Root Cause:** Relational databases like PostgreSQL spawn a dedicated backend OS process (or thread) per connection. Each connection consumes memory (~2MB–10MB) and CPU for context switching.
* **Serverless / Auto-Scaling Amplification:** If 100 auto-scaled application instances spawn, each opening a connection pool of 20 connections, 2,000 active connections bombard the database, causing memory starvation, connection refusal errors (`FATAL: sorry, too many clients already`), and system crashes.
* **Solution:** **Database Connection Pooling** (e.g., PgBouncer, HikariCP, or AWS RDS Proxy).
* A lightweight proxy layer sits between servers and the database. It maintains a small, fixed multiplexed pool of persistent connections to the database engine (e.g., 50 connections) while accepting thousands of incoming ephemeral client requests.



```
App Instances (100s) ──► [ Connection Pooler / PgBouncer ] ──► Database (Fixed ~50 connections)

```

### Problem 3.2: Heavy Read Volumes

* **The Root Cause:** Most Web/Mobile apps exhibit an asymmetric **Read-to-Write ratio** (often 90:10 or 99:1—e.g., viewing feeds/profiles vs. posting updates). A single primary database handling both reads and writes runs out of IOPS and CPU capacity.
* **Prerequisite Step (Non-Architectural):** Ensure missing **B-Tree indexes** are added to query filter columns (`WHERE email = ...`, `JOIN` foreign keys). An unindexed query scanning 10 million rows looks like a hardware bottleneck when it is actually an indexing flaw.
* **Solution:** **Primary-Replica (Leader-Follower) Replication**.
* **Primary (Writer):** Handles 100% of mutations (`INSERT`, `UPDATE`, `DELETE`, transactions).
* **Replicas (Readers):** Asynchronously pull transaction logs (WAL/binlog) from the Primary and serve read queries (`SELECT`).



```
                    ┌──► [ Read Replica 1 ] (Read-only)
                    │
App Servers ────────┼──► [ Read Replica 2 ] (Read-only)
  │ (Writes)        │
  ▼                 └──► [ Read Replica 3 ] (Read-only)
[ Primary DB ] ──(Async WAL Sync)──┘

```

* **Costs & Trade-offs:**
* **Replication Lag (Eventual Consistency):** Replication is typically asynchronous to prevent write performance penalties on the Primary. Network bottlenecks or heavy write bursts cause replicas to lag milliseconds to seconds behind the Primary.
* **Read-Your-Own-Writes Consistency Issue:** A user updates their user profile bio (write hits Primary). The app redirects them to view their updated profile (read hits Replica). Because of 500ms replication lag, the page renders the *old bio*, confusing the user.
* *System Workaround:* Implement **Sticky Reads** or **Write-Through Routing**—force reads from the Primary for a short time window (e.g., 5 seconds) after a user performs a write action, or read user-critical data directly from the Primary while sending public feed reads to Replicas.



---

## Phase 4: Caching Expensive Computations

* **The Problem:** Certain queries execute complex joins, aggregations, or scans over millions of rows (e.g., counting total followers, calculating trending items, computing site metrics). Re-running these queries thousands of times per second degrades DB performance and wastes computing resources.
* **Solution:** **Distributed In-Memory Cache** (e.g., **Redis** or **Memcached**).
* Data is stored in key-value pairs inside RAM rather than reading from disk/SATA/NVMe storage, offering sub-millisecond retrieval speeds.



### Cache Lookup Strategies

1. **Cache-Aside (Lazy Loading):**
* App checks Redis for key `user:profile:42`.
* **Cache Hit:** Return cached data directly to user.
* **Cache Miss:** Query database, write result to Redis key with an Expiration Time (TTL), and return data.



```
Request ──► [ App ] ──1. Check Cache──► [ Redis Cache ]
             │                             │ (Hit: Return Data)
             ├──2. Miss: Read DB─────────► [ Database ]
             └──3. Write to Cache────────► [ Redis Cache ]

```

2. **Write-Through / Write-Back:**
* **Write-Through:** App writes to cache and DB synchronously.
* **Write-Back (Write-Behind):** App writes instantly to cache; cache asynchronously persists to DB in batches (high throughput, risk of data loss if cache crashes before flush).



### Cache Invalidation & Trade-offs

* **The Hard Problem:** "There are only two hard things in Computer Science: cache invalidation and naming things." (Phil Karlton).
* **Cache Staleness:** When database records update, cached entries become invalid until expired or purged.
* **Failure Modes:**
* **Thundering Herd / Cache Stampede:** A heavily requested cache key expires. Hundreds of concurrent requests experience a cache miss simultaneously and hit the database at the exact same millisecond, overloading the DB.
* *Fix:* Use Mutex Locks (single request computes DB query, others wait) or Probabilistic Early Expiration (XFetch).


* **Cache Penetration:** Requests for non-existent keys repeatedly bypass the cache and hit the DB.
* *Fix:* Cache null values or use **Bloom Filters**.




* **Business Logic Rule:** Cache selection is a product trade-off decision. Metric counters, media feeds, and catalog items can tolerate 30-second stale reads. Financial ledger balances, stock inventory counts, and authentication credentials should **never** be cached carelessly without strong invalidation.

---

## Phase 5: Asynchronous Processing (Message Queues & Background Workers)

* **The Problem (Synchronous Blocking & Cascading Failures):**
* When a user performs an action (e.g., Sign-up, Order Checkout), the application server may need to:
1. Insert user into DB (20ms)
2. Call Third-Party Email API to send Welcome Email (800ms)
3. Generate Welcome PDF receipt (1500ms)
4. Fire Analytics event to external tracker (300ms)


* Total response time: ~2.6 seconds. The user is stuck watching a spinner.
* **Coupled Vulnerability:** If the external email API experiences an outage, your entire signup flow throws an HTTP 500 error, blocking users from registering even though database user creation succeeded.


* **Solution:** **Asynchronous Message Queues & Background Processing**.
* Decouple heavy, non-critical computations and external network calls into background jobs using message queues (e.g., RabbitMQ, Apache Kafka, Amazon SQS, or Redis-backed queues like BullMQ/Celery).



```
User Request
     │
     ▼
[ App Server ] ──1. Write DB Record ──► [ Database ]
     │
     ├──2. Enqueue Job ("send_email", userId) ──► [ Message Queue (Redis / SQS) ]
     │
     └──3. Instant HTTP 200 Response to Client
                                                        │
                                                        ▼
                                             [ Background Worker ]
                                             (Pulls job & calls Email API)

```

* **Workflow:**
1. App server creates the database record synchronously.
2. App server publishes a job payload `{ task: "SEND_EMAIL", userId: 42 }` to the queue.
3. App server immediately responds with `HTTP 200/201 Success` to the client (Total latency: ~25ms).
4. An independent pool of **Background Workers** pops jobs off the queue asynchronously, executing emails, analytics, and PDF processing.


* **Costs & Trade-offs:**
* **At-Least-Once Delivery & Idempotency:** Message queues usually guarantee "at-least-once" delivery. Workers may process the same job twice due to network retries. Worker job handlers **must be idempotent** (e.g., check `payment_processed_id` before charging a card twice).
* **Dead Letter Queues (DLQ):** Failed jobs (e.g., malformed payloads, repeated API failures) must be routed to a DLQ for manual inspection or retry backoff strategies (Exponential Backoff + Jitter).



---

## Phase 6: Scaling Data Volume (Database Sharding)

* **The Problem:** Read Replicas duplicate 100% of the dataset on every replica node. When total data volume reaches multi-terabytes or millions of writes per second, no individual database instance has enough disk capacity, RAM, or IOPS to store the monolithic dataset.

```
Monolithic DB (10 TB Data) ───► EXCEEDS SINGLE DISK / NODE LIMITS

```

* **Solution:** **Database Sharding (Horizontal Partitioning)**.
* Splitting a monolithic database into multiple smaller, independent database instances called **Shards**. Each shard contains a distinct subset of the overall data.



```
                  [ Router / Application Layer ]
                                │
       ┌────────────────────────┼────────────────────────┐
       ▼                        ▼                        ▼
[ Shard 1 ]              [ Shard 2 ]              [ Shard 3 ]
(Users 1-1,000,000)      (Users 1,000,001-2M)     (Users 2,000,001-3M)

```

### Partitioning Mechanics & Shard Keys

* **Shard Key Selection:** A critical column present in queries used to route requests to the correct shard (e.g., `user_id`, `tenant_id`).
* **Routing Algorithms:**
1. **Modulo / Hash-Based Sharding:**

$$\text{Shard ID} = \text{Hash}(\text{Shard Key}) \pmod{\text{Total Shards}}$$


* *Example:* `Hash(user_id=105) % 3 = Shard 3`.


2. **Range-Based Sharding:** Routing based on defined key ranges (e.g., User IDs 1–1M $\rightarrow$ Shard 1, 1M–2M $\rightarrow$ Shard 2). Risk: Creates "hotspots" on recent ranges receiving all new write traffic.
3. **Directory-Based Sharding:** A lookup service maintains a mapping table of `Key -> Shard ID`.



### The True Costs & Pain Points of Sharding

* **Cross-Shard Queries (Scatter-Gather):** Queries that lack the Shard Key (e.g., `SELECT * FROM users WHERE created_at > '2026-01-01'`) must query *every single shard in parallel*, gather the sub-results over the network, and aggregate/sort them in application memory. High latency and CPU cost.
* **Loss of Referential Integrity & Joins:** Cross-shard SQL `JOIN` operations and Foreign Key constraints spanning multiple databases are physically impossible or prohibitively slow. Data relationships must be denormalized or joined in application code.
* **Resharding & Rebalancing Complexity:** Adding a 4th shard to a Modulo-3 system changes the denominator from `% 3` to `% 4`. Up to 75% of existing keys suddenly compute a new Shard ID, requiring massive dataset migrations across network nodes while keeping the app online.
* *Mitigation:* **Consistent Hashing** algorithms to minimize key movements during node additions/removals.



---

## Summary Architecture Checklist for Engineering Notes & Interviews

| Architectural Bottleneck | Primary Symptom | Solution | Introduced Component | Key Trade-off / Cost |
| --- | --- | --- | --- | --- |
| **CPU / RAM Starvation** | Server response times spike, request queues fill up & time out. | Horizontal Scaling | Load Balancer (e.g., NGINX, ALB) | Requires completely stateless application servers; SPOF risk. |
| **Session Loss across Nodes** | Users randomly get logged out on navigation clicks. | Centralized Session Store | Redis / Memcached | Extra sub-millisecond network hop per request; authentication dependency on Redis. |
| **DB Connection Overhead** | `Too many connections` errors during traffic spikes. | DB Connection Pooling | PgBouncer / RDS Proxy | Connection limit capping; overhead proxy management. |
| **DB Read Exhaustion** | High CPU / Disk IOPS on primary DB due to heavy `SELECT` traffic. | Read Replicas (Primary-Replica) | Read-Only Replicas | Replication Lag (Eventual consistency); stale reads. |
| **Repeated Expensive Queries** | High DB CPU re-calculating complex joins/aggregations. | In-Memory Caching | Redis Cache | Cache invalidation complexity; risk of stale product data. |
| **UI Blocked by Side Effects** | Slow page loads due to external APIs (email, PDF generation). | Asynchronous Task Queue | Message Queue & Background Workers | Non-instant execution; requires idempotent handlers & retry policies. |
| **Single DB Disk / IO Limits** | Dataset size exceeds maximum disk capacity of a single machine. | Database Sharding | Sharding / Partitioning Layer | Loss of ACID cross-shard joins; costly cross-shard scatter-gather queries. |


## Source

- [YouTube video](https://youtu.be/EaXHfuHRWwg?si=h7hQv4g64lgEXt3X)