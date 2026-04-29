---
name: orchestrator
description: >
  Master orchestrator for code and architecture auditing — coordinates 'meaningful-reasoning',
  'code-auditor', 'system-auditor', and 'caveman' into a lean 5-step pipeline: preflight →
  run audits → findings gate → apply changes → brief final summary. Trigger on any audit, review, bug-find, remediation, or refactor
  request involving code or architecture, especially multi-file projects; prefer this over
  calling sub-skills directly.
---

# Orchestrator

Central orchestration brain. Does **not** audit or fix directly. Routes, delegates, synthesizes.

**Internal agent language:** wenyan-ultra (see caveman skill). All internal reasoning
and agent→orchestrator summaries in wenyan-ultra. Translate to English for user output only.

---

## Pipeline Overview

```
[0] Preflight  →  [1] Run Audits  →  [2] Findings Gate (await approval)  →  [3] Apply  →  [4] Summary
```

Steps 0–2 always complete before pausing for user input.  
Steps 3–4 execute only after explicit approval.

---

## Mode Selection (Do This First)

Classify the request before anything else.  
Legend: **⚡** = Fast-path · **S** = Standard · **D** = Deep

**Tie-break rule (top-to-bottom; first match wins):**

| Priority | Situation | Mode |
|---|---|---|
| 1 | User says "quick" or "just check X" | **⚡** relevant skill only — skip the other |
| 2 | Single function ≤ 30 lines AND no system description | **⚡** code-auditor direct |
| 3 | One architecture dimension only AND no code | **⚡** system-auditor direct |
| 4 | Multi-file / file tree / > 150 lines total | **D** full shard + pipeline |
| 5 | Only code, no system description | **S** code-auditor only |
| 6 | Only system design, no code | **S** system-auditor only |
| 7 | Code + system both provided | **S** parallel agents |

**Check Constraint-Aware Routing before finalising mode** — user constraints override.

**⚡ Fast-path:** emit one `<wc>` mode decision, call the relevant sub-skill directly.  
Pass through the sub-skill's output unchanged. **Do NOT use the 6-step pipeline for ⚡.**

```
<wc>⚡：單函數9行→代碼審官快徑。</wc>
```

---

## Step 0 — Preflight

Read `/mnt/skills/user/meaningful-reasoning/SKILL.md` and run it silently.

Apply the Causal Chain: does this audit request have a traceable outcome? Is the scope well-formed — code to review, a system to evaluate, a clear remit? Are there hidden proxy goals the audit might not serve?

If the chain is missing, ask **one** clarifying question before spawning any agents. Do not explain the preflight framework to the user.

**For ⚡ Fast-path:** run meaningful-reasoning inline — do not spawn as a subagent. Complete the preflight, then proceed directly to the relevant sub-skill.

**When invoked via orchestrator**, meaningful-reasoning runs once at the pipeline level — code-auditor and system-auditor do not re-run it independently.

---

## Step 1 — Run Audits

### 1a — Scope Inventory

```
<wc>
  審：代碼+架構俱在。代碼審官+架構審官。
  邊界：代碼審官←實作；架構審官←系統設計。
</wc>
```

State one-sentence task boundary per agent before spawning.

### 1b — Parallel Agent Calls (Pass 1)

| Agent | Scope | Returns |
|---|---|---|
| `code-auditor` | Implementation: bugs, design, security, tests | Issue triples, tier labels |
| `system-auditor` | Architecture: boundaries, data flow, tech fit, scale, ops, security | Issue triples, dimension labels |

Use the agent prompt templates at the bottom of this skill.  
Do NOT ask agents to comment on each other's domain.

**Monitor for interrupt signals during Pass 1.** If either agent emits `CRITICAL_INTERRUPT`:

```
🚨 CRITICAL INTERRUPT
════════════════════
Agent: <code-auditor|system-auditor>
Finding: <description>  Severity: CRITICAL

  • "address now" — pause scan, fix immediately, then resume
  • "log and continue" — elevate to top of Findings Gate, finish scan
  • "halt" — stop all agents
```

Pause and show this gate before continuing. Do not proceed until response received.

### 1c — Cross-Agent Signal Exchange (Pass 2)

After Pass 1, extract cross-cutting signals in a `<wc>` block, then run targeted re-queries:

```
<wc>跨信：代碼←auth超時→架構auth層薄弱。架構←無速率限制→代碼API端點裸露。</wc>
```

| If you find… | Re-query target |
|---|---|
| Code finding implying system root cause | `system-auditor` cross-signal prompt |
| System finding implying code exposure | `code-auditor` cross-signal prompt |

Merge Pass 2 responses into the findings set.

### 1d — Synthesize

Translate wenyan-ultra → English (internal). Then:

1. **Dedup** — architectural root cause beats implementation symptom:
   ```
   <wc>重複：auth超時←代碼+架構。根因在架構→保架構，棄代碼症狀。</wc>
   ```
2. **Cross-reference** — flag code issues that indicate system design problems:
   ```
   <wc>跨維：代碼auth問題→系統授權層薄弱。升級跨切面。</wc>
   ```
3. Assign IDs: `[C-NN]` = code · `[S-NN]` = system · `[X-NN]` = cross-cutting.

---

## Step 2 — Findings Gate ⛳

Present a **compact** summary. No verbose solutions yet — those come after approval.

```
FINDINGS SUMMARY
════════════════════
Scope: <what was audited>
Mode: <S|D>   Agents: <code-auditor | system-auditor | both>
Issues: <N code · M architecture · K cross-cutting>

── CRITICAL / HIGH ─────────────────────────────────────────────
  [C-01] [Tier]   <one-line description>    owner: code
  [S-01] [Dim]    <one-line description>    owner: arch
  [X-01]          <one-line description>    owner: both

── MEDIUM / LOW ────────────────────────────────────────────────
  [C-02] [Tier]   <one-line description>
  [S-02] [Dim]    <one-line description>

Approve:
  • "proceed"         — apply all fixes
  • "skip [ID]"       — exclude items (e.g. "skip C-02, S-02")
  • "only [ID,ID]"    — apply only listed items
  • "details"         — show full findings before deciding
  • "cancel"          — stop here
```

**Wait for explicit user response before Step 3.**

- `proceed` → apply all items
- `skip [ID]` → remove those items; re-show the gate
- `only [ID,ID]` → scope application to those items
- `details` → output full ORCHESTRATION REPORT (see Reference Format), then re-show the gate
- `cancel` → stop; do not apply anything

**Do not output anything until the user responds.**

---

## Step 3 — Apply Changes

For each approved item, delegate to the appropriate sub-skill in **Apply Mode**.

### Code fixes → code-auditor Apply Mode

Read `/mnt/skills/user/code-auditor/SKILL.md`.  
Pass approved `[C-NN]` and `[X-NN]` IDs + the original code.  
Request: generate fixed code + per-issue diff summary.

```
汝：代碼審官，應用模式。
已批：[C-NN]…（附原碼）
任：生成修正後完整代碼+每問題單行變更摘要。
格式：修正後代碼塊 + 變更摘要列表。
禁：新問題，新重構。唯修已批。
```

### Architecture fixes → system-auditor Apply Mode

Read `/mnt/skills/user/system-auditor/SKILL.md`.  
Pass approved `[S-NN]` IDs + original system description.  
Request: generate actionable implementation steps (first step must be executable).

```
汝：架構審官，應用模式。
已批：[S-NN]…
任：每問題生成具體實施步驟（首步可執行）。
格式：[S-NN] 標題 + 步驟列表。
禁：重複問題描述。唯輸行動。
```

Present results as they arrive — do not wait for both to complete.

---

## Step 4 — Brief Final Summary

After all apply calls complete:

```
DONE
════════════════════
Applied:  [C-01], [S-01], [X-01]
Skipped:  [C-02]
Files:    <changed files, or "N/A — architecture guidance only">

Next:  <one sentence on highest-remaining risk, or "No outstanding issues.">
```

Do not repeat issue descriptions. Do not add commentary. Stop here.

---

## Deep Pipeline (D mode) — Steps A–F

> Use for multi-file / > 150 lines. After A–F complete, feed findings into the standard Findings Gate (Step 2) and continue.

### Step A — Ingest

Map the full file tree. Tag: `Core` (types/interfaces/configs/base) · `Leaf` (everything else).

### Step B — Shard

Divide files into slices **< 100 lines**. IDs: `[FILE_ID]:[SHARD_N]`.

### Step C — Enrich

Prepend each shard with a Global Context Header (imported types, parent signatures, exported constants).  
Pruning: strip inline comments, dead code, console.log, redundant whitespace. Retain type defs, signatures, constants.

### Step D — Self-check (orchestrator inline)

| Check | Pass | Fail |
|---|---|---|
| Types resolved | All types present in header | Re-enrich; if still unresolved, mark `Context Insufficient`, skip |
| Imports present | No unresolved references | Re-enrich; if still unresolved, flag + skip |
| Shard auditable | Contains real logic | Mark `Empty After Pruning`, skip silently |

### Step E — Execute (→ code-auditor)

Pass only shards that cleared all Step D checks.  
Directive: "Fix logic errors and style issues. Maintain all interface signatures and exported names."

### Step F — Track State

```
<audit-state>types.ts:1:C:✓ | auth:3:C:⟳ | fmt:2:L:…</audit-state>
```

Format: `file:N:T:S` — N=shard count, T=C(ore)/L(eaf), S=✓/⟳/…/✗/⏸  
On `HALT` mid-scan: freeze state, append checkpoint, go to Findings Gate immediately.

---

## Circuit Breaker Rules

| File Type | Failure | Action |
|---|---|---|
| Any | `Context Insufficient` | Re-enrich; if unresolved, skip + log |
| Leaf | code-auditor fails 2× | Flag `Logic Error`, skip, continue |
| Core | code-auditor fails any | **HALT** — show gate immediately |

```
⛔ ORCHESTRATION HALTED
Core file [FILE_ID] failed at Step E.
Reason: [brief reason]
Action required: [what user needs to clarify]
```

---

## Interrupt & Checkpoint Protocol

| Signal | Source | Action |
|---|---|---|
| `HALT: FULL_REFACTOR` | system-auditor | Freeze code-auditor. Mark `⏸`. Go to Findings Gate. |
| `HALT: CRITICAL_SEC` | either agent | Pause. Show CRITICAL INTERRUPT gate. Await user. |

**Technical Debt mode** (when `FULL_REFACTOR` confirmed):
> "Catalog only. List all issues with file, line, and tier. Do NOT generate fix recommendations."

**Resume after canceled trigger:** restore checkpoint, mark `⏸` → `⟳`, continue from saved shard.

---

## Constraint-Aware Routing

Check BEFORE mode selection — constraints override.

| Phrase | Override |
|---|---|
| "freeze", "deploy in Xh", "no time for full audit" | **High-Severity Only** |
| "technical debt only" | code-auditor **Technical Debt mode** |
| "security only" | Skip all non-security tiers/dimensions |

Note constraint in Findings Gate header:
```
Constraint: HIGH_SEVERITY_ONLY  (production freeze 48h)
```

---

## Agent Prompt Templates

### code-auditor — Pass 1
```
汝：代碼審官。唯審實作，禁論架構拓撲。
域：設計｜正確｜安全｜測試。
碼：
<paste verbatim>
審畢。問題：<issue>｜風：<risk>｜策：<fix>
無問題：<clean tiers>
限十。
發現嚴重漏洞：立發 CRITICAL_INTERRUPT｜<issue>｜<class>。
```

### system-auditor — Pass 1
```
汝：架構審官。唯審系統，禁論行級代碼。
域：整體設計｜資流｜技選｜擴性｜運維｜安全。
系：
<paste verbatim>
審畢。問題：<issue>｜風：<risk>｜策：<fix>
無問題：<clean dimensions>
限八。跨維衝突標：跨維：<dim>。
判需全重構：發 HALT: FULL_REFACTOR｜<reason>。判嚴重安全：發 HALT: CRITICAL_SEC｜<issue>。
```

### code-auditor — Pass 2 cross-signal
```
汝：代碼審官。架構審官發現跨信：
[S-NN] <signal in wenyan-ultra>
問：此架構缺陷在代碼中有何暴露點？標引用ID [S-NN]。
唯答暴露點，禁重複架構議題。限三。格式：[C-NN]←[S-NN]｜暴露：<location>｜風：<risk>
```

### system-auditor — Pass 2 cross-signal
```
汝：架構審官。代碼審官發現跨信：
[C-NN] <signal in wenyan-ultra>
問：此代碼問題是否反映架構根因？指明架構層何處。標引用ID [C-NN]。
唯答架構根因，禁重複代碼症狀。限三。格式：[S-NN]←[C-NN]｜根因：<layer>｜策：<fix>
```

---

## Reference Format (on "details" reply)

```
ORCHESTRATION REPORT
════════════════════
Scope: <what was reviewed>  Mode: <S|D>

── CODE AUDIT ──────────────────────────
[C-01] [Tier]: <description>
Solution: <precise fix>

── SYSTEM AUDIT ────────────────────────
[S-01] [Dimension]: <description>
Risk: <risk>
Recommendation: <actionable fix>

── CROSS-CUTTING ───────────────────────
[X-01] [C-NN] → [S-NN]: <code symptom of system-level problem>
Fix: <unified resolution>

── CLEAN ───────────────────────────────
<tiers/dimensions with no issues>

── EXECUTION GRAPH ─────────────────────
Mode: <S|D>  Constraints: <none|…>
Pass1:  code-auditor ∥ system-auditor  signals: <or: none>
Pass2:  <cross-signal agents, or: skipped>
Gates:  Findings Gate · CRITICAL gate (if triggered)
```

Omit empty sections. Issue IDs must match those in the Findings Gate.

---

## Escalation Signals

| Signal | Action |
|---|---|
| Code issue is symptom of arch root cause | → `[X-NN]` in Findings Gate |
| `HALT: FULL_REFACTOR` | → Findings Gate immediately; code-auditor → Technical Debt mode |
| > 6 architectural issues | → standalone system-auditor deep dive; halt code-auditor batches |
| > 15 code issues | → warn in Findings Gate before applying |
| Security at both code AND system level | → top of Findings Gate; `HALT: CRITICAL_SEC` path |

---

## Startup Checklist

1. Confirm sub-skill files accessible:
   - `/mnt/skills/user/meaningful-reasoning/SKILL.md`
   - `/mnt/skills/user/code-auditor/SKILL.md`
   - `/mnt/skills/user/system-auditor/SKILL.md`
   - `/mnt/skills/user/caveman/SKILL.md` (D mode output)
2. Check constraints (Constraint-Aware Routing)
3. Classify request → select mode (⚡ / S / D)
4. If **D**: request file tree / code files if not provided
5. Begin `<wc>` analysis; output initial `<audit-state>` before first batch (D only)

---

## Reference

- meaningful-reasoning → `/mnt/skills/user/meaningful-reasoning/SKILL.md`
- code-auditor → `/mnt/skills/user/code-auditor/SKILL.md`
- system-auditor → `/mnt/skills/user/system-auditor/SKILL.md`
- wenyan-ultra → `/mnt/skills/user/caveman/SKILL.md`
- Never use wenyan-ultra for user-facing output unless user requests it
