# Audit Tier Definitions

Read this file when you need the full taxonomy for a Standalone Audit — especially for
Tier 1 where named anti-patterns and architectural frames are useful to cite precisely.
For Self-Audit, the checklists in SKILL.md are sufficient.

---

## Tier 1 — Design Integrity (full taxonomy)

- **Spaghetti code** — flag codebases where control flow cannot be traced linearly without holding large amounts of external state in your head:
  - Tangled call graphs — function A calls B calls C calls back into A with no clear coordinator; execution paths cross in ways that make reasoning about state impossible
  - Deep action-at-a-distance — a change in one function silently changes observable behaviour in a far-removed, unrelated function
  - Shared mutable state used as an informal message-passing system — modules communicate by mutating globals or singletons rather than through defined interfaces
  - No clear entry/exit contracts — callers cannot know what a function expects or guarantees without reading its full implementation
  - Files or classes that exist as "catch-all" bins — utility files that accumulate unrelated helpers, or classes that expand to absorb every feature that didn't fit elsewhere
  - *The test:* if you cannot explain what a 100-line block does in two sentences without listing every branch, it is spaghetti.

- **Design patterns — misuse and missed opportunities** — flag when patterns are applied incorrectly, or when the absence of a well-known pattern is causing real pain:
  - *Anti-patterns to flag by name:*
    - **God Object / God Class** — a single class or module that knows or does too much; the system cannot change without touching it
    - **Anemic Domain Model** — domain objects are plain data bags; all logic lives in service layers that reach into them, inverting encapsulation
    - **Feature Envy** — a function/method that obsessively accesses data from another class instead of living there
    - **Primitive Obsession** — raw strings, ints, or booleans standing in for domain concepts that deserve their own types (e.g., `userId: string` everywhere instead of a `UserId` type)
    - **Shotgun Surgery** — a single conceptual change requires edits scattered across many unrelated files
    - **Parallel Inheritance Hierarchies** — adding a subclass in one hierarchy forces adding one in another
    - **Magic Numbers / Strings** — unexplained literals embedded in logic with no named constant or enum
  - *Pattern misuse to flag:*
    - A Singleton used to avoid passing a dependency — this is dependency hiding, not design
    - A Factory or Abstract Factory with only one concrete product — YAGNI (flag under Overengineering too)
    - Observer/event systems wired so loosely that side-effects are untraceable — favour explicit calls unless fan-out is genuinely dynamic
    - Strategy pattern implemented but the strategy is always the same one — dead flexibility
  - *Missed pattern opportunities (flag only when the pain is real and present, not hypothetical):*
    - Repeated `if/else` or `switch` on a type tag that will clearly grow — suggest Strategy or polymorphism
    - Object construction logic scattered across the codebase — suggest Factory or Builder
    - Resource acquisition/release duplicated everywhere — suggest a context manager, RAII wrapper, or using/with construct

- **Architectural frame of reference** — when evaluating system-level or multi-file code, calibrate issues against established structural models. You do not need to prescribe a specific architecture, but use these as a lens to articulate *why* a structural issue is a problem:
  - *Layered architecture* — UI / API → Service / Application → Domain → Infrastructure. Flag when lower layers import upper ones, or when domain logic appears in the infrastructure layer (e.g., business rules inside a SQL query or ORM callback).
  - *Hexagonal / Ports-and-Adapters* — the application core should be ignorant of how it is driven (HTTP, CLI, queue) and what it drives (DB, email, third-party APIs). Flag when HTTP request objects or ORM models bleed into the domain.
  - *Clean Architecture / Dependency Rule* — dependencies should point inward toward policy; nothing in the core should know about the outermost ring. Flag inward rings importing outward ones.
  - You are not auditing for architectural purity. Use these frames to name the *direction* of a coupling problem when it would otherwise be hard to articulate.

- **Modularity & single responsibility** — every function, class, and module should do one thing. Flag when:
  - A function has more than one reason to change (e.g. it validates input *and* persists *and* sends a notification)
  - Mixed abstraction levels within a function — a low-level I/O call sitting next to a high-level business rule in the same block
  - A class or module accumulates unrelated responsibilities over time (god class / god module)
  - A function is only safely callable in a specific sequence — hidden temporal coupling that belongs in a coordinator

- **Readability & control flow** — code should read top-to-bottom without the reader mentally simulating state. Flag:
  - *Arrow code* — deeply nested conditionals (3+ levels) that can be flattened with early returns or guard clauses
  - *Long functions* — a function that can't be understood without scrolling; break at natural seams into named helpers
  - *Boolean flag parameters* — `process(data, true, false)` tells the reader nothing; replace with named functions or an options object/struct
  - *Negative conditionals* — `if !isNotReady` compounds cognitive load; invert to the positive form
  - *Implicit control flow* — exceptions used for non-exceptional paths; callbacks buried inside callbacks; early returns that silently skip contract enforcement

- **Overengineering (YAGNI)** — flag complexity added for hypothetical future needs that don't exist yet:
  - Abstraction layers with a single concrete implementation that will never be swapped
  - Interfaces, base classes, or generics introduced before there are two meaningfully different cases
  - Plugin systems, registries, or factory hierarchies where a simple function call would suffice
  - Over-parameterized functions (5+ parameters, or parameters that only exist to support untested hypothetical cases)
  - Configuration keys or feature flags that gate behaviour that will never actually vary
  - When a case doesn't match the above exactly: flag it if the complexity added exceeds the present requirements — not anticipated future ones

- **DRY violations** — flag duplicated *logic*, not just duplicated text:
  - The same conditional guard repeated across multiple call sites instead of being enforced once at the source
  - Copy-pasted blocks that diverge subtly over time — the worst bugs live here
  - Parallel data structures that must be kept in sync manually (two collections representing the same domain concept)
  - Any logic where a single rule change would require updates in more than one location — that's the DRY violation

- **Separation of concerns & decoupling** — flag:
  - Business logic mixed into I/O handlers (route handlers doing domain work, event listeners making DB calls directly)
  - Presentation logic mixed with data transformation
  - Law of Demeter violations — reaching through an object to call methods on its internals (`a.getB().getC().doThing()`)
  - Tight coupling to concrete types where a simpler abstraction would decouple the dependency

- **Reversibility** — hard-wired major dependencies (DB, framework, auth provider, storage backend) that can't be swapped without a rewrite; vendor lock-in leaking into business logic

- **Design by Contract** — missing preconditions / postconditions / invariants; silent acceptance of invalid input; state-corrupting failure paths

- **Testability** — flag structural reasons a unit can't be tested in isolation:
  - A function that constructs its own dependencies internally instead of accepting them — the dependency can't be replaced in a test without modifying the function
  - Direct access to global or shared state (singletons, module-level variables, static fields) — tests can't control or isolate the state
  - I/O mixed with logic in the same function — the logic can't be tested without performing the I/O
  - If writing the simplest test requires mocking more than two dependencies, the function is too coupled — split it

  *(Flag the structural issue here; write the concrete missing test cases in Tier 4)*

- **Naming** — names should make the reader not need a comment. Flag:
  - Cryptic abbreviations (`usr`, `tmp`, `val`, `mgr`) with no domain meaning
  - Misleading names — a function named `getUser` that also creates one; a boolean named `flag`
  - Single-letter variables outside of loop indices or well-established math conventions
  - Inconsistent vocabulary — the same concept called `user`, `account`, and `member` in different parts of the codebase

- **Comments & maintainability** — flag bad comments and missing good ones.

  **Bad comments (flag and remove):**
  - *Restate the code* — `i++ // increment i`, `// call the login function` above `login()`. If reading the code is enough, the comment adds noise.
  - *Document a fix or change* — `// fixed null pointer`, `// added check per PR #42`, `// was previously X`. These are version-control messages — they belong in commit history, not source. They will rot and mislead.
  - *Temporary notes left in production* — `// TODO: clean this up`, `// hack for now`, `// not sure why this works`. Either resolve them or track them in an issue tracker. Ownerless TODOs are dead weight.
  - *Verbose paraphrasing* — multi-line prose that repeats what the next three lines of code clearly show. Length does not equal value.

  **Good comments (flag when missing on non-trivial code):**
  - *The why behind a non-obvious decision* — "We skip cache here because this endpoint is called post-write and stale reads cause visible data loss."
  - *Constraints and invariants that aren't visible in the type system* — "Callers must hold the write lock before calling this."
  - *Business rules embedded in logic* — "Accounts created before 2021-01-01 use the legacy fee schedule."
  - *Intentional omissions* — "We do not retry here — the upstream service is not idempotent."
  - *Gotchas and known traps* — "This library silently truncates strings > 255 chars; we cap input before this point."
  - *Public API docstrings* — every exported function/class should have a one-line summary of what it does and what it returns. Parameters and error cases for non-obvious signatures.

- **Infrastructure & architectural coupling** — flag design choices that bake deployment or infrastructure concerns into business logic:
  - *Hardwired infrastructure* — flag any value that would need to change when deploying to a different environment: connection strings, hostnames, queue names, bucket paths, region identifiers. These belong in config/env injection, not source code.
  - *Service boundary violations* — business logic that directly reaches across service or module boundaries (e.g. one service calling another's DB, one module constructing another's internals). Each service/module should own its data and expose a contract.
  - *API contract hygiene* — missing versioning strategy, implicit coupling on response shape, no clear error contract. Flag when callers are expected to parse raw error messages or internal field names.
  - *Scalability assumptions baked in* — code that assumes a single instance (in-process state, local filesystem paths used as shared storage, non-atomic counters). Flag when this would silently break under horizontal scaling.
  - *Configuration / environment separation* — mixing dev/staging/prod behaviour in the same code path (e.g. `if env == "prod"` branching). Configuration should drive behaviour; code should not.
  - *Data ownership and coupling* — two modules sharing a mutable data structure by reference, or a module that reads/writes another module's "owned" tables directly. Flag the ownership gap and suggest an interface.
  - *Schema and model design* — missing indexes on queried columns, nullable fields used as implicit booleans, polymorphic columns that break query integrity, no clear ownership of schema migrations.

---

## Tier 2 — Correctness (full taxonomy)

- **Bugs / logic errors** — off-by-ones, wrong operators, incorrect branching, silent failures
- **Error handling** — bare excepts, unhandled rejections, missing null checks
- **Edge cases** — empty / zero / max / concurrent inputs not handled
- **Concurrency** — shared mutable state, missing locks, non-atomic operations
- **Resource management** — unclosed handles, missing context managers, leaked connections
- **Performance** — flag measurable inefficiencies, not speculative micro-optimizations; also flag premature optimization that adds complexity before a bottleneck is proven:
  - Algorithmic complexity — an O(n²) pattern where an O(n) approach exists (nested loops over the same collection, repeated linear searches in a hot path)
  - Redundant I/O — fetching the same data multiple times within a single request/operation instead of fetching once and passing it down
  - N+1 query patterns — querying inside a loop when a single batched query would suffice
  - Blocking calls in async paths — synchronous I/O or CPU-heavy work on an event loop thread
  - Unnecessary allocations in hot paths — creating intermediate collections or objects that are immediately discarded
  - Missing caching for expensive, pure, or rarely-changing computations
  - Premature optimization — flag optimization that: adds indirection without a measured bottleneck, uses non-obvious data structures without a documented reason, or makes code materially harder to read for gains that aren't proven necessary
- **Observability** — missing logs on failure paths, no correlation IDs, insufficient error context
- **Config & deps** — hardcoded values that belong in env/config; unpinned dependency versions; unused imports

---

## Tier 3 — Security (full taxonomy)

- **Injection** — SQLi, command injection, template injection, XSS, XXE
- **Auth & authorisation** — missing auth checks, privilege escalation, insecure session handling, JWT misconfiguration
- **Broken access control** — missing ownership checks, IDOR
- **Sensitive data exposure** — secrets or PII in logs, error responses, or source
- **Hardcoded credentials** — API keys, passwords, tokens, private keys
- **Input validation** — unsanitised user input reaching a sink (filesystem, DB, shell, network)
- **Insecure defaults** — permissive CORS, disabled TLS verification, debug mode in production, world-readable permissions
- **Vulnerable dependencies** — known-CVE library versions, risky transitive deps
- **Path traversal & SSRF** — user-controlled paths or URLs passed to filesystem/network without validation

---

## Tier 4 — Tests (full taxonomy)

Suggest missing test cases that would meaningfully increase confidence. Cover in order, skip what doesn't apply:

- **Happy path** — primary success scenario per public function / endpoint
- **Edge cases** — empty, zero, max, boundary, type-coercion inputs
- **Error paths** — each explicit exception or error return path; verify the caller receives the correct signal (error type, code, or message) and cannot silently ignore the failure
- **Security inputs** — malformed tokens, oversized payloads, injected characters, concurrent writes
- **Integration / contract** — cross-module boundary behaviour

**Calibrate to context:** Production → all categories. Internal tool → happy path + edges + errors. Prototype → happy path + critical errors only. Throwaway → `No suggestions.`
