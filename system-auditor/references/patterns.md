# Architectural Patterns Reference

Quick reference for named patterns — when to use, when to avoid, and the signal that suggests each.

---

## Decomposition Patterns

### Strangler Fig
**Use when:** Migrating a monolith to services incrementally without a big-bang rewrite.
**How:** Route traffic through a facade; move functionality slice by slice behind new services; retire monolith sections as they're replaced.
**Avoid when:** The monolith's data model is so entangled that no slice can be cleanly extracted — fix the data model first.
**Signal:** "We need to modernize but can't stop shipping."

### Anti-Corruption Layer (ACL)
**Use when:** Integrating with a legacy system or third-party API whose model is incompatible with your domain model.
**How:** Introduce a translation layer that maps the external model to your internal concepts. The rest of your system never sees the foreign model.
**Avoid when:** The external model is simple and stable — an ACL adds indirection without benefit.
**Signal:** "We keep having to work around what this external system gives us."

### Backend for Frontend (BFF)
**Use when:** Different clients (mobile, web, third-party) have meaningfully different data and interaction needs from the same backend.
**How:** One dedicated API surface per client type, each aggregating and shaping data optimally for that client.
**Avoid when:** Client needs are largely identical — a shared API with query parameters is simpler.
**Signal:** "Mobile needs a different response shape than web and we keep adding special cases."

---

## Data Consistency Patterns

### Saga
**Use when:** A business transaction spans multiple services and must remain consistent without distributed locks.
**Choreography variant:** Each service emits events; others react. Simple, low coordination overhead. Gets hard to reason about as the chain grows.
**Orchestration variant:** A central coordinator (saga orchestrator) directs each step. Easier to reason about; the orchestrator is a coordination bottleneck.
**Avoid when:** The transaction touches a single service's data — a local transaction is correct and simpler.
**Signal:** "We need atomic-ish behavior across services without 2PC."

### Outbox Pattern
**Use when:** A service must both write to its database and publish an event, and these must be atomic (no phantom events, no lost events).
**How:** Write the event to an `outbox` table in the same transaction as the business data. A relay process reads the outbox and publishes to the message bus.
**Avoid when:** At-least-once delivery with idempotent consumers is acceptable — the outbox adds operational overhead.
**Signal:** "We sometimes publish events that never committed, or commit data without publishing the event."

### CQRS (Command Query Responsibility Segregation)
**Use when:** Read and write patterns are so different that a single model serves neither well; or read scale is 10×+ write scale.
**How:** Separate the write model (commands, domain logic) from the read model (query-optimized projections, potentially a different store).
**Avoid when:** The read/write ratio is balanced and the domain is simple. CQRS adds two models to maintain.
**Signal:** "Our reads need denormalized views but our writes need normalized integrity — we can't have both with one model."

### Event Sourcing
**Use when:** Complete audit trail is a core requirement; temporal queries ("what did the state look like on date X?") are needed; or event replay is valuable for projections.
**How:** Store every state change as an immutable event. Current state is derived by replaying events.
**Avoid when:** Querying current state is the primary workload and audit trails are not required. Event sourcing significantly increases read complexity.
**Signal:** "We need to know not just what state is now, but how it got there."

---

## Integration Patterns

### Circuit Breaker
**Use when:** Calling an external service that may degrade or fail; you need to prevent cascading failure.
**How:** Track recent failure rate. When it exceeds a threshold, open the circuit (fail fast without calling the dependency). Half-open after a timeout to probe recovery.
**Avoid when:** The dependency is local in-process — overhead without benefit.
**Signal:** "When the payment service is slow, everything else piles up."

### Bulkhead
**Use when:** One workload type consuming all resources would starve another (e.g., a slow batch job starving API requests).
**How:** Isolate resource pools (thread pools, connection pools, queue partitions) per workload class.
**Signal:** "One class of traffic makes the whole system slow for everyone."

### Idempotent Consumer
**Use when:** Processing messages from a queue or event stream that guarantees at-least-once delivery.
**How:** Track a unique message/event ID. Skip (or return success for) already-processed IDs.
**Signal:** "Duplicate messages are causing double-processing."

### Competing Consumers
**Use when:** Processing throughput needs to scale horizontally on a queue.
**How:** Multiple consumer instances pull from the same queue; each message processed once (relies on queue's visibility timeout / ack model).
**Signal:** "One consumer can't keep up with the queue depth."

---

## Infrastructure Patterns

### Sidecar
**Use when:** Cross-cutting concerns (auth, mTLS, logging, tracing) should be consistent across services without each service implementing them.
**How:** A sidecar container runs alongside each service, handling the cross-cutting concern transparently (e.g., Envoy in a service mesh).
**Avoid when:** You have 2–5 services — sidecar overhead and complexity isn't worth it at small scale.
**Signal:** "Every service is re-implementing the same auth/logging/retry logic."

### API Gateway
**Use when:** Multiple services are exposed externally and you need centralized auth, rate limiting, routing, and request aggregation.
**Avoid when:** Internal service-to-service calls — an API gateway between internal services is usually an anti-pattern (adds latency and a central failure point).
**Signal:** "We need to enforce auth and rate limiting consistently across all public-facing APIs."

### Service Mesh
**Use when:** You have many services (10+) with complex east-west traffic, need mutual TLS, distributed tracing, and fine-grained traffic management.
**Avoid when:** Fewer than ~10 services — the operational overhead of a service mesh exceeds its benefit.
**Signal:** "We can't reason about what's talking to what, and we can't enforce consistent policies."

---

## When Patterns Are the Wrong Answer

Patterns solve specific problems. Before applying one, confirm:
1. The problem it solves is actually the problem you have
2. The trade-offs it introduces are ones you're willing to accept
3. Your team has the operational capability to run it correctly

**Anti-pattern:** applying a pattern because it's modern or impressive (event sourcing for a CRUD app, microservices for a two-person team, CQRS for read-heavy but simple data). The result is accidental complexity that compounds over time.
