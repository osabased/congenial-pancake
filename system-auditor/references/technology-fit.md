# Technology Fitness Criteria

Guidance for evaluating whether a technology choice is the right fit for a given system context. Use this when auditing technology selections or advising on greenfield choices.

---

## How to Evaluate Fitness

For each technology in scope, answer:

1. **Fit signal:** Does the technology's core strengths match the workload's core requirements?
2. **Mismatch signal:** Does the workload lean heavily on the technology's known weaknesses?
3. **Operational cost:** What does running this at the team's current scale and maturity actually cost?
4. **Replaceability:** How hard is it to swap this out in 18 months if requirements shift?

A technology may be a good fit on criteria 1 and 2 but a poor choice on 3 or 4 for a specific team. Fitness is always contextual.

---

## Message Queues & Event Streaming

### Kafka
**Strong fit:** High-throughput event streaming, durable event log, event sourcing replay, log compaction use cases, fan-out to many consumers.
**Weak fit:** Simple task queues with few producers/consumers, small message volumes, teams without Kafka operational experience.
**Operational cost:** High. Kafka clusters require careful partition management, retention policy, and monitoring. Managed offerings (Confluent, MSK) reduce but don't eliminate this.
**Watch for:** Using Kafka as a task queue (correct tool is SQS/RabbitMQ); using it for request/response patterns (wrong model entirely).

### SQS / RabbitMQ / Similar Task Queues
**Strong fit:** Task distribution, work queues, competing-consumer patterns, simple pub/sub, retry with DLQ.
**Weak fit:** Event replay, event sourcing, high-fanout streaming to many independent consumers (each needs its own queue).
**Watch for:** Treating SQS as an event log (messages are deleted on consume — no replay).

### Redis Pub/Sub
**Strong fit:** Low-latency ephemeral messaging, real-time notifications where message loss is acceptable.
**Weak fit:** Reliable delivery — Redis pub/sub offers no persistence, no consumer groups, no replay. If the consumer is offline, messages are lost.
**Watch for:** Using Redis pub/sub where message delivery guarantees are actually required.

---

## Databases

### PostgreSQL / RDBMS
**Strong fit:** Relational data with complex query patterns, strong consistency requirements, transactions across multiple entities, audit trails, well-understood schema.
**Weak fit:** Write-heavy time-series workloads, schema-less document storage at very high write throughput.
**Watch for:** Modeling document-style data as many narrow JSONB columns (lose query-ability); using as a message queue (works but doesn't scale well past low volumes).

### MongoDB / Document Stores
**Strong fit:** Flexible/evolving schema, document-shaped data, developer-speed priority, read-heavy workloads with denormalized documents.
**Weak fit:** Complex relational queries (joins across collections are painful), strong multi-document transactional requirements, teams that need schema enforcement at the DB layer.
**Watch for:** Using documents to model relational data (you'll rebuild the relational model in application code); ignoring schema validation entirely (schema-on-read becomes schema-chaos).

### DynamoDB / Key-Value / Wide-Column
**Strong fit:** Known access patterns, high-scale read/write throughput, single-digit millisecond latency, infinite scale-out.
**Weak fit:** Ad hoc queries, data exploration, evolving access patterns, complex reporting.
**Watch for:** Designing the table before knowing all access patterns (DynamoDB is access-pattern-first); using a scan as a fallback (table scans are expensive and slow).

### Redis (as a primary store)
**Strong fit:** Cache, ephemeral session data, leaderboards, counters, rate limiting, pub/sub.
**Weak fit:** Durable primary store for data you cannot afford to lose (AOF/RDB persistence helps but Redis is not a database replacement for critical data).
**Watch for:** Treating Redis as the source of truth without a persistence layer beneath it.

### Time-Series Databases (InfluxDB, TimescaleDB, Prometheus)
**Strong fit:** Metrics, sensor data, event logs where time is the primary query axis, high write throughput of narrow rows.
**Weak fit:** General-purpose relational workloads, complex joins across entities.
**Watch for:** Storing time-series data in a general-purpose RDBMS at high volume (query performance and storage costs both suffer).

### Search Engines (Elasticsearch, OpenSearch)
**Strong fit:** Full-text search, faceted filtering, log aggregation and analysis, geospatial queries.
**Weak fit:** Primary transactional store (not ACID, eventual consistency model, schema migrations are painful), source of truth (treat as a secondary index).
**Watch for:** Elasticsearch as the primary data store — it should be derived from a durable source via indexing pipeline.

---

## Communication Protocols

### REST / HTTP
**Strong fit:** Public APIs, CRUD resources, human-readable payloads, wide client compatibility, stateless interactions.
**Weak fit:** High-performance internal service-to-service calls with complex schemas (gRPC is faster and schema-enforced); streaming data.
**Watch for:** REST for request/response patterns that are fundamentally bidirectional or streaming (use WebSockets or SSE instead).

### gRPC
**Strong fit:** Internal service-to-service calls where performance and schema enforcement matter, polyglot environments with generated clients, streaming RPCs.
**Weak fit:** Public APIs consumed by browsers (requires a proxy layer), teams unfamiliar with Protobuf toolchain, simple CRUD resources.
**Watch for:** gRPC chosen for public APIs without a transcoding proxy (browser support is limited without grpc-web or REST transcoding).

### GraphQL
**Strong fit:** APIs consumed by multiple clients with different data needs (reduces over/under-fetching), flexible schema evolution, BFF use cases.
**Weak fit:** Simple APIs with stable, uniform data needs (REST is simpler); backends serving single-purpose internal services.
**Watch for:** N+1 query problems in resolvers without a DataLoader-style batching layer; exposing a monolithic GraphQL API that hits 10+ downstream services (latency amplification).

### WebSockets / SSE
**Strong fit:** Real-time bidirectional communication (WebSockets), server-to-client event streaming (SSE).
**Weak fit:** Simple request/response patterns (HTTP is better), fire-and-forget notifications (WebSockets have higher per-connection overhead).

---

## Caches

### CDN Caching
**Strong fit:** Static assets, geographically distributed delivery, response caching for public content.
**Weak fit:** User-specific or frequently invalidated content.

### Application-Level Cache (Redis, Memcached)
**Strong fit:** Reducing database load for read-heavy, predictable queries; session storage; rate limiting.
**Weak fit:** Primary data store; data that changes with high frequency (invalidation overhead exceeds benefit).
**Watch for:** Caching inconsistency — ensure cache invalidation is triggered on write, not on TTL expiry alone for consistency-sensitive data.

---

## Compute & Hosting

### Containers / Kubernetes
**Strong fit:** Multiple services requiring independent scaling, polyglot environments, teams with container operational maturity.
**Weak fit:** A single service run by a small team — managed platform (App Runner, Cloud Run, Fly.io, Railway) is simpler and cheaper.
**Watch for:** Kubernetes as the default for everything — it has significant operational overhead that must be justified by scale or multi-service complexity.

### Serverless (Lambda, Cloud Functions)
**Strong fit:** Infrequent or bursty workloads, event-driven pipelines, low operational overhead priority.
**Weak fit:** Long-running processes, workloads with high cold-start sensitivity, teams that need predictable pricing at constant high volume.
**Watch for:** Cold starts affecting latency-sensitive user-facing paths; Lambda functions that embed complex orchestration logic (use Step Functions or a workflow engine instead).

### Managed Services vs. Self-Hosted
**Default to managed** unless: (a) cost at your scale makes self-hosted economically justified, or (b) managed offerings don't meet your compliance requirements.
Operational burden of self-hosting databases, message queues, and search engines is consistently underestimated at early stages.

---

## Framework & Language Fitness Signals

| Signal | Flag |
|---|---|
| Choosing a language because the team knows it | Fine — team velocity matters |
| Choosing a language for a workload it's structurally bad at | Flag (e.g., Python for CPU-bound numeric without numpy/native extensions) |
| Mixing 4+ languages across a small team | Flag — operational and hiring cost exceeds flexibility benefit |
| Framework chosen for its hype cycle, not its feature fit | Flag — assess the actual problem it solves vs. alternatives |
| Using a framework that predates the team's ability to upgrade it | Flag — version lag compounds into security and maintenance debt |
