---
name: code-auditor
description: "Production code auditor with two modes: Standalone Audit (review user-provided code, produce a structured remediation report) and Self-Audit (audit your own generated code before presenting it). Trigger on any code review, audit, bug report, security check, 'fix my code', 'refactor', 'clean up', 'code smell', 'improve quality', or 'critique' request — and whenever you generate non-trivial code (more than 20 lines of real logic)."
---

# Code Auditor

---

## Modes of Operation

| Mode | When | Output |
|---|---|---|
| **Standalone Audit** | User provides code and asks for review, critique, audit, bugs, refactor, or improvement | Structured remediation report; no rewrite |
| **Self-Audit** | You just generated non-trivial code (>20 lines, meaningful logic) | Audit your draft, fix all issues, present improved code + fix summary |
| **Apply** | Orchestrator delegates approved issue IDs post-gate | Fixed code block + per-issue change summary |

Simple one-liners, boilerplate stubs, and illustrative pseudocode do not need Self-Audit. Standalone Audit supports parallel subagent fan-out; Self-Audit is sequential (draft → critique → fix) and must not use subagents.

---

## Apply Mode

Triggered by the orchestrator after user approves specific issue IDs at the Findings Gate.

**Inputs received:** original code (verbatim) + list of approved issue IDs with their descriptions.

**Rules:**
- Fix **only** the approved issues. Do not introduce new changes, refactors, or style improvements.
- Maintain all interface signatures, exported names, and public API contracts.
- If fixing one approved issue requires touching code adjacent to another approved issue, batch the edits cleanly — do not leave inconsistent state.
- If an approved fix is impossible without breaking a contract, output a one-line blocker note for that ID and skip it.

**Output format:**

```
── APPLY: <filename> ───────────────────────────────────────────

```<language>
[complete fixed file content]
```

── CHANGES ─────────────────────────────────────────────────────
[C-01] <one-line description of what changed and where>
[C-02] <one-line description>
[X-01] <one-line description>

── BLOCKERS (if any) ───────────────────────────────────────────
[C-NN] <reason fix cannot be applied without breaking contract>
```

Omit the BLOCKERS section if there are none. Do not repeat issue descriptions from the audit — the change summary is a log of *what was done*, not a re-statement of the problem.

---

## Output Rules (both modes)

**Standalone Audit report:**
- Open `<thinking>` first: trace control flow, evaluate all four tiers, apply Tier Boundary Rule. If you spend more than two sentences deciding tier placement, apply the rule and move on.
- After `</thinking>`: exactly one fenced code block (plus an optional context-assumption sentence). No preamble, no commentary, no sign-off outside the block.
- First line inside the block: exact filename(s), or `[untitled — <inferred purpose>]`.
- Multi-file: prefix every issue with `[filename]`; cross-file issues get `[cross-file]`.
- All four tier headings always present, even when clean (`No issues found.` / `No suggestions.`).

**Self-Audit summary** (after the improved code block):
- Format: `Fixed (N issues): • [Tier] fix` then `Tests: <gaps or "No gaps found.">` — see Self-Audit Mode for the full template.
- Omit a tier from `Fixed` if it had no issues. Never invent fixes.
- Tone: transparent log. No "robust", "production-ready", or "secure".
- **Zero-issue skepticism:** If `<self_correction>` finds zero issues for code over 50 lines, re-read the Design checklist once from the top before concluding clean. A genuinely zero-issue result on complex code is rare. Never invent issues — only scrutinize harder.

**Never:** conversational filler before/after the report; restating the user's code; explaining tier headings; apologizing for issues found.

---

## Pre-Audit: Preflight & Dependency Check (both modes)

### Preflight

**When invoked standalone** (not via orchestrator): read `/mnt/skills/user/meaningful-reasoning/SKILL.md` and run it silently before proceeding. Confirm there is a traceable outcome — code to review, a scope, a clear remit. If the chain is missing, ask one clarifying question. Do not mention the preflight to the user.

**When invoked via orchestrator**: orchestrator runs meaningful-reasoning at Step 0. Skip the independent preflight here — do not re-run it.

### Dependency Check

Scan for references to external scripts, modules, or files not in context (imports, shell invocations, file paths, dynamic loading, out-of-context function calls).

**If found:** follow `references/dependency-check.md`. **If none:** proceed — do not mention this check.

In subagent mode: orchestrator performs this check before spawning tier subagents; halt and surface the notice if unresolved.

---

## Standalone Audit Mode

### Subagent Orchestration

#### Single-File: Parallel Tier Fan-Out

Spawn three tier subagents simultaneously:

| Subagent | Tier |
|---|---|
| `agents/ca-tier-design.md` | Design Integrity |
| `agents/ca-tier-correctness.md` | Correctness |
| `agents/ca-tier-security.md` | Security |

Each receives: full code (verbatim), inferred context, detected language.

**Tests tier:** Orchestrator evaluates after collecting tier results — testability assessment benefits from seeing the full Design and Correctness picture first.

**Synthesis:**
1. Resolve all `POSSIBLE OTHER TIER:` flags via Tier Boundary Rule; drop the flag.
2. Deduplicate cross-tier findings — keep the earlier-tier instance.
3. Apply length calibration; trim to highest-severity if combined count exceeds expected range.
4. Assemble final report.

#### Multi-File: Parallel Per-File Fan-Out

Spawn one `ca-file-reviewer.md` subagent per file. Each receives: its file + filename, inferred context, detected language, list of other filenames.

**Synthesis:**
1. Collect `CROSS-FILE:` flags; consolidate duplicates into one cross-file issue.
2. Apply Tier Boundary Rule to any cross-tier flags.
3. Deduplicate within-file findings that duplicate a cross-file finding — keep the `[cross-file]` version.
4. Assemble report; prefix all issues with `[filename]` or `[cross-file]`.

### Fast Path (trivial code — no subagents)

Single function, ≤ ~30 lines, one clear job: audit all four tiers inline. When in doubt, use Fast Path — subagent overhead exceeds audit time for small code.

### Fallback: Sequential Mode (no subagents)

Load `references/audit-tiers.md`, then audit all four tiers yourself in order. Same checklists, output format, and quality rules apply.

---

## Context Awareness

Infer intent from context clues and calibrate strictness:

| Context | Approach |
|---|---|
| Production / deployed | Full audit — all categories |
| Internal tool / team script | Full audit — skip YAGNI and speculative design |
| Prototype / proof-of-concept | Focus on bugs, security, structural red flags; note but don't labour over style |
| Personal / throwaway | Bugs and security only; write `No suggestions.` for Design non-bug and Tests |

If ambiguous, default to full audit and state assumption in one sentence before the report block.

### Language-Specific Adaptation

| Language | Key idioms |
|---|---|
| Python | List comprehensions vs. loops, context managers, f-strings, type hints, `__all__`, Pythonic iteration |
| JavaScript / TypeScript | `async`/`await` vs. raw Promises, `const`/`let` discipline, null coalescing, strict typing |
| Go | Error-as-value, goroutine lifecycle (termination condition required), channel direction types, `select` with `default` vs blocking, `sync.WaitGroup` misuse (`Add` inside goroutine), `defer` in loops, interface-based design |
| Java / Kotlin | Checked exceptions, `AutoCloseable` lifecycle, nullability annotations |
| Rust | Ownership, borrow checker intent, `unwrap()` usage, `?` propagation |
| SQL / DDL | Column type correctness (INT vs BIGINT, TEXT vs VARCHAR), CHECK constraints, index presence/selectivity, FK enforcement, UNIQUE for idempotency keys, partition strategy. Constraint/index issues → code-auditor; entity-relationship and service boundary questions → system-auditor. |
| Terraform / k8s YAML | Resource limits/requests, image tag pinning (no `latest`), secret management (no inline plaintext), RBAC least-privilege, probe correctness, namespace isolation, deprecated API versions |

Flag language-idiomatic issues under the most appropriate tier (usually Design or Correctness).

### Output Format

```
auth.py

--- Design ---

Issue 1: <concise description of the problem and where it occurs>
Solution: <precise, implementable fix — exact names, replacements, or logic>

--- Correctness ---

Issue 2: <concise description>
Solution: <precise fix>

--- Security ---

Issue 3: <concise description>
Solution: <precise fix>

--- Tests ---

Test 1: <scenario — e.g. "call login() with an expired token">
Assert: <expected outcome — e.g. "raises AuthError with code 401, session is not created">
```

Each test suggestion must name the function under test, describe the input/state, and state the exact expected outcome (return value, exception, side-effect, or observable behaviour).

### Quality Bar for Solutions

Each Solution must answer: **what** to change (specific line/block/pattern), **how** to change it (replacement code, logic, or approach), **why** only if it prevents ambiguity.

Bad: "Add error handling."
Good: "Wrap the `subprocess.run()` call on line 42 in try/except catching `subprocess.CalledProcessError` and `FileNotFoundError`; log with `logger.error()` and re-raise or return a non-zero exit code."

---

## Self-Audit Mode

No subagents. Draft → critique → fix executes in a single context window.

### Workflow

**Step 1 — Draft**
Write your initial implementation inside `<draft_code>`. Mandatory — the draft must exist as tokens for the critique step to evaluate.

```
<draft_code>
[initial implementation]
</draft_code>
```

**Step 2 — Critique**
Open `<self_correction>` immediately after. Run checklists against the draft. Evaluate what the code *does*, not what you *intended*. Quote or reference specific lines from `<draft_code>` for every finding — vague references to "the function" or "the loop" are not permitted.

```
<self_correction>
[Design]
  - <quoted line or construct> — <why this violates the checklist>

[Correctness]
  - <quoted line or construct> — <why this is a bug or gap>

[Security]
  - <quoted line or construct> — <why this is a risk>

[Tests]
  - <gap description> — <what scenario is untestable or missing>
</self_correction>
```

Write `none` for any tier with no genuine issues — do not omit tiers. Do not soften or hedge findings.

**Design checklist:**
- Single responsibility per function — no validate + persist + notify in one block
- No mixed abstraction levels — low-level I/O not adjacent to high-level business logic
- No temporal coupling — no function only safely callable in a specific sequence
- No long functions — if it requires scrolling, split into named helpers
- No arrow code — conditionals not nested 3+ levels
- No negative/double-negative conditionals — `if !isNotReady` must be inverted
- No boolean flag parameters — `fn(data, true, false)` is absent
- No spaghetti — control flow traceable linearly; no tangled call graphs, no action-at-a-distance, no shared mutable globals as informal message-passing
- No anti-patterns — no God Object, Anemic Domain Model, Shotgun Surgery, Primitive Obsession, Feature Envy
- No overengineering — no abstraction layer, interface, or factory without two real present use cases
- No DRY violation — no logic change requiring updates in more than one location
- No misleading or cryptic names
- No comment restating the code or documenting a fix; only write comments when *why* is non-obvious

**Correctness checklist:**
- Null / empty / zero inputs handled explicitly
- Every error path propagates or handles — nothing swallowed
- No resource opened without guaranteed close path
- No N+1 — no query or I/O in a loop where a single batched call would suffice
- No premature optimization — no added indirection without a measured bottleneck

**Security checklist:**
- Every user-supplied value sanitised before reaching a sink (DB, shell, filesystem, network)
- No hardcoded secret, credential, or token
- No missing auth or ownership check on any operation that modifies or exposes data

**Tests checklist:**
- Primary happy-path test for each public function is obvious to write without accessing internal state
- If the simplest test requires mocking >2 dependencies, flag the function for a Design issue — too coupled to test cleanly

**Step 3 — Fix**
Close `<self_correction>`, then apply fixes for every real issue listed. Fix Design first — it often eliminates downstream Correctness and Security problems. Issues requiring no code change (e.g., test gaps for the caller) go into the Self-Audit Summary, not silently dropped.

**Step 4 — Present**
Output the final improved code in a standard markdown code block, then the Self-Audit Summary. Every finding from `<self_correction>` must appear as a fix made or a noted gap — nothing disappears silently.

### Self-Audit Output Format

```
<draft_code>
[initial implementation]
</draft_code>

<self_correction>
[Design]
  - <finding or "none">

[Correctness]
  - <finding or "none">

[Security]
  - <finding or "none">

[Tests]
  - <finding or "none">
</self_correction>

```python
[final improved code]
```

Self-Audit Summary
──────────────────
Fixed (N issues):
  • [Design] <one-line description>
  • [Correctness] <one-line description>
  • [Security] <one-line description>

Checked:
  ✓ Design
  ✓ Correctness
  ✓ Security
  ✓ Tests
Clean: <comma-separated tiers with no issues, or "all tiers">

Tests: <"No gaps found." or brief list of test cases the caller should add>
```

When `<self_correction>` finds nothing (all tiers `none`): use `Fixed: none.` / `Clean: all tiers` / `Tests: No gaps found.`

`<draft_code>` and `<self_correction>` must always be present — they are proof the audit ran. A Self-Audit Summary without them above it is a hallucinated summary.

---

## When to Escalate to System-Auditor

Code-auditor operates at the **micro level** — functions, modules, files. When conversation involves the following, also trigger (or recommend) **system-auditor** for a macro-level review:

| Signal | Example |
|---|---|
| Multiple services or bounded contexts | "We have an API gateway, auth service, and worker queue" |
| Technology/framework selection | "Should we use Kafka or SQS?" |
| Integration and data-flow design | "How should these two systems communicate?" |
| Architectural fitness for scale/team | "Is this monolith fine, or split it up?" |
| Cross-system reliability or operational concerns | "What happens when the payment service is down?" |
| Greenfield or major refactor decisions | "We're re-platforming — does this design make sense?" |

Do not merge the two audits. Run code-auditor first.

---

## Thread A/B Workflow (Self-Audit blind spot)

For generated logic over **~100 lines**, Self-Audit carries its *intent* in context and audits charitably. Fix: wipe that intent entirely.

**Subagent variant (preferred when available):** Spawn `agents/ca-fresh-reviewer.md` with only the code — no conversation history.

**Thread A/B fallback:**
1. **Thread A** — Generate the code. Do not audit here.
2. **Thread B (new conversation)** — Paste the code with no additional context; trigger Standalone Audit Mode: *"Audit this code."*

Use whenever the code touches production, handles untrusted input, or runs unattended.

---

## Review Scope & Issue Ordering

Assess in tier order: **Design → Correctness → Security → Tests.** Design first — fixing it often eliminates downstream bugs and security problems.

**Tier Boundary Rule:** When an issue fits two tiers, assign to the *earlier* one immediately. If you've written more than two sentences weighing placement, apply the rule and move on.

| Looks like… | Goes in… | Reason |
|---|---|---|
| Missing null check / bare except | **Correctness** | It's a bug, not a structural smell |
| Error handling structurally absent across a whole layer | **Design** | Root cause is an architectural gap |
| Hardcoded secret | **Security** | Primary risk is exposure, not a logic bug |
| SQL built by string concatenation | **Security** | Injection risk outweighs style |
| Design by Contract violation that also causes a bug | **Design** | Fix the contract; the bug disappears |

In subagent mode: tier subagents flag cross-tier candidates with `POSSIBLE OTHER TIER:` rather than silently dropping them; orchestrator resolves during synthesis.

Full tier taxonomy: `references/audit-tiers.md`. In subagent mode: pre-loaded into tier agents.

---

## Length Calibration (Standalone Audit)

Judge scope by distinct functions/classes, logical responsibilities, and files — not line count:

| Scope | Label | Expected issues |
|---|---|---|
| Single function or small script, one clear job | **Trivial** | 1–5. Only what genuinely matters. Use Fast Path. |
| Several functions / one module, one main purpose | **Small** | 5–15. Cover all tiers proportionally. |
| Multiple modules or files, several responsibilities | **Medium** | 15–30. Cross-file patterns and design debt should appear. |
| System-level: many files, architectural decisions visible | **Large** | 30+. Prioritize systemic patterns over one-off instances. |

If the report feels long relative to visible scope, add inside the code block after all issues:

    [Note: Issue volume suggests systemic problems — consider a broader refactor rather than line-by-line remediation.]

In subagent mode: apply length calibration during synthesis, not inside tier subagents.

---

## Agent Files Reference

| File | Purpose |
|---|---|
| `agents/ca-tier-design.md` | Single-file parallel — Design tier |
| `agents/ca-tier-correctness.md` | Single-file parallel — Correctness tier |
| `agents/ca-tier-security.md` | Single-file parallel — Security tier |
| `agents/ca-file-reviewer.md` | Multi-file parallel — full four-tier review per file |
| `agents/ca-fresh-reviewer.md` | Fresh-eyes review — no intent carried over |
| `references/audit-tiers.md` | Full Design tier taxonomy |
| `references/dependency-check.md` | Dependency notice template and branching logic |

## Skill References

- meaningful-reasoning → `/mnt/skills/user/meaningful-reasoning/SKILL.md` (preflight, standalone invocation only)
- orchestrator → `/mnt/skills/user/orchestrator/SKILL.md` (coordinates this skill at pipeline level)
