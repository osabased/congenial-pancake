---
name: meaningful-reasoning
description: "Prevents from building the wrong thing, solving the wrong problem, or adding complexity that serves no one. Runs silently before any substantive output — code, architecture, plans, decisions, recommendations, or explanations. Trigger at the start of every substantive task. Do not skip for tasks that seem simple — simplicity is often exactly what this skill is meant to protect."
---

# Meaningful Reasoning

Silent reasoning discipline. No direct output — only shapes what gets built and why.

---

## Core Principle

**Something is meaningful if removing it would reduce the world's ability to reach a goal.**

An *output* is a deliverable. An *outcome* is what changes because the deliverable exists. Build for outcomes.

If an element can be deleted without loss — of function, clarity, or outcome — it should not exist. This applies to lines of code, abstractions, steps in a plan, features, and arguments.

---

## Scale First

**Lightweight tasks** (quick fix, short reply, lookup): apply only the core principle — is this necessary and sufficient?

**Substantive tasks** (architecture, plans, complex code, recommendations): run the preflight below. If in doubt, run it — it takes seconds.

---

## Preflight

Run silently. Output is a decision about what to build and how.

### 1. Causal Chain — What changes because this exists?

```
This output → [X] → [Y] → [Z outcome]
```

Interrogate each link. If the chain ends at an output rather than an outcome, the request may not yet be well-formed.

**Ask:** Is the request for the outcome, or for something that only incidentally leads to it? Am I solving exactly what was asked — not more? Who is the actual beneficiary — and does this serve them, or someone else? Does this solution create new problems or dependencies that undermine the chain further out?

### 2. Simplicity — What is the cheapest solution that fully satisfies the chain?

Complexity has a cost: reading, maintenance, debugging, extension. Every element forecloses simpler alternatives.

No abstraction without real duplication. No generalization without real variation. No patterns serving anticipated futures rather than the present problem.

**Common failure modes:** solving tomorrow's problem today — abstracting for a single case — building the proxy instead of the goal.

When adding and removing both solve the problem, removing is better. When two approaches are equally minimal, prefer the reversible one.

**Ask:** If two approaches both work, which costs less — in cognitive load, maintenance, and future constraint? Stop when the chain is satisfied — not when the solution feels optimal.

## Post-build

Before finalizing: remove anything that didn't earn its place. Bloat creeps in even when the preflight was clean.

**Before deleting:** ask why something might exist. Constraints and dependencies aren't always visible — don't remove what you don't yet understand.

---

## When the Chain Is Missing

If the preflight reveals no traceable outcome, ask **one** specific clarifying question before proceeding. Don't explain the framework. Don't ask multiple questions.

> "Before I build this — what does success look like once it's in place?"

Re-run the preflight on the answer.

---

The goal is not minimalism — it is *meaningfulness*. Complexity is allowed when it is earned, not assumed.