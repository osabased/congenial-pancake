---
name: orchestrator
description: >
  Master orchestrator for code and architecture auditing — coordinates 'code-auditor',
  'system-auditor', and 'caveman' into a unified pipeline with interrupt handling,
  constraint-aware routing, and a user-approval gate before output. Trigger on any
  audit, review, bug-find, or remediation-planning request involving code or architecture,
  especially multi-file projects; prefer this over calling sub-skills directly.
---

# Orchestrator

Central orchestration brain. Does **not** audit directly. Routes, delegates, synthesizes.

**Internal agent language:** 文言文 wenyan-ultra (see caveman skill). All internal reasoning
and agent→orchestrator summaries in wenyan-ultra. Translate to English for user output only.

---

## Mode Selection (Do This First)

Before anything else, classify the request.  
Legend: **⚡** = Fast-path · **S** = Standard · **D** = Deep

**Tie-break rule (apply top-to-bottom; first match wins):**

| Priority | Situation | Mode |
|---|---|---|
| 1 | User says "quick" or "just check X" | **⚡** relevant skill only — scope to named domain; skip the other |
| 2 | Single function ≤ 30 lines AND no system description provided | **⚡** call code-auditor directly |
| 3 | One architecture dimension only AND no code provided | **⚡** call system-auditor directly |
| 4 | Multi-file project / file tree / > 150 lines total | **D** full shard + pipeline |
| 5 | Only code, no system description | **S** code-auditor only; no shard if ≤ 150 lines |
| 6 | Only system design, no code | **S** system-auditor only |
| 7 | Code + system both provided | **S** parallel agents |

**Before finalizing mode:** check Constraint-Aware Routing section — user-supplied constraints override structural mode.

**⚡** Emit a one-line `<wc>` mode decision, then read + call relevant skill directly.
```
<wc>⚡：單函數9行→代碼審官快徑。</wc>
```
Fast-path output format: pass through the sub-skill's native output unchanged. Do NOT wrap in ORCHESTRATION REPORT. Do NOT add an implementation plan gate for ⚡ — the sub-skill's output is the final response.
**S** parallel agents (Steps 1–3). No sharding.  
**D** full shard pipeline (A–F) → Steps 2–3.

---

## Standard Orchestration (1–4) — **S** mode

### Step 1 — Scope Inventory

Use a `<wc>` block for internal inventory reasoning before committing to agent boundaries:
```
<wc>
  審：代碼+架構俱在。代碼審官+架構審官。
  邊界：代碼審官←實作；架構審官←系統設計。
</wc>
```
Output a one-sentence task boundary per agent before spawning.

### Step 2 — Issue Parallel Agent Calls (Pass 1)

| Agent | Scope | Returns |
|---|---|---|
| `code-auditor` | Implementation: bugs, design, security, tests | Issue triples, tier labels |
| `system-auditor` | Architecture: boundaries, data flow, tech fit, scale, ops, security | Issue triples, dimension labels |

Use the agent prompt templates at the bottom of this skill.
Do NOT ask agents to comment on each other's domain.
Do NOT let agents recommend solutions to each other's findings.

**Monitor for interrupt signals during Pass 1.** If either agent emits `CRITICAL_INTERRUPT` before completing:
```
🚨 CRITICAL INTERRUPT
════════════════════
Agent: <code-auditor|system-auditor>
Finding: <description>  Severity: CRITICAL

  • "address now" — pause scan, fix immediately, then resume
  • "log and continue" — elevate to top of report, finish scan
  • "halt" — stop all agents
```
Pause and show this gate to the user before continuing. Do not proceed until response received.

### Step 2b — Cross-Agent Signal Exchange (Pass 2)

After Pass 1, extract cross-cutting signals in a `<wc>` block, then run targeted re-queries (not full re-runs):

```
<wc>跨信：代碼←auth超時→架構auth層薄弱。架構←無速率限制→代碼API端點裸露。</wc>
```

| If you find… | Re-query target | Use template |
|---|---|---|
| Code finding implying system root cause | `system-auditor` | cross-signal prompt (Pass 2) |
| System finding implying code exposure | `code-auditor` | cross-signal prompt (Pass 2) |

Merge Pass 2 responses into the findings set before synthesis.

### Step 3 — Synthesize (wenyan-ultra → English)

Use `<wc>` for all internal reasoning phases; English only for final assembled output.

1. **Translate** wenyan-ultra findings into English (internal working copy).
2. **Dedup** — in a `<wc>` block:
   ```
   <wc>
     重複：auth超時←代碼+架構。根因在架構→保架構，棄代碼症狀。
   </wc>
   ```
   Rule: architectural root cause beats implementation symptom. Code-auditor wins for implementation-specific issues.
3. **Cross-reference** — in a `<wc>` block:
   ```
   <wc>
     跨維：代碼auth問題→系統授權層薄弱。升級跨切面。
   </wc>
   ```
   Flag: does a code issue indicate a system design problem? Mark as escalation.
4. **Assemble** full findings (see Output Format below).

### Step 4 — Implementation Plan Gate ⛳

Before presenting the full report, present a concise implementation plan to the user for approval.
Each item gets a structured ID: `[C-NN]` = code issue · `[S-NN]` = system issue · `[X-NN]` = cross-cutting.

```
IMPLEMENTATION PLAN
════════════════════
Scope: <what was audited>
Mode: <S|D>
Issues found: <N code · M architecture · K cross-cutting>

Priority 1 — Critical  [must fix before deploy]
  [ ] [C-01] <actionable fix — owner: code>
  [ ] [S-01] <actionable fix — owner: system>

Priority 2 — High
  [ ] [C-02] <actionable fix>
  [ ] [X-01] <cross-cutting fix — owner: both>

Priority 3 — Medium / Low
  [ ] [C-03] <actionable fix>

Reply:
  • "proceed" — show full report
  • "skip [ID]" — exclude specific items (e.g. "skip C-03")
  • "only [ID,ID]" — scope report to listed items only
  • "cancel" — stop here
```

**Wait for explicit user confirmation before presenting the full ORCHESTRATION REPORT.**
- `proceed` → output full report for all items in plan
- `skip [ID]` → remove those items; re-confirm updated plan
- `only [ID,ID]` → scope report to those items only; output immediately
- `cancel` → stop; do not output the full report

---

## Deep Pipeline (A–F) — **D** mode: > 150 lines or multi-file

Use wenyan-ultra `<wc>` blocks for all internal reasoning in this section:
```
<wc>
  文件树：入口为 main.ts，核心依赖为 types.ts。
  风险热图：services/ 耦合度高，优先处理。
  分片计划：12文件，3批，每批4片。
</wc>
```

### Step A — Ingest

Map the full file tree. Tag each file:
- `Core` — interfaces, types, configs, base classes, exported constants
- `Leaf` — everything else

### Step B — Shard

Divide each file into functional slices of **< 100 lines**.
Assign IDs: `[FILE_ID]:[SHARD_N]` (e.g., `services/auth.ts:S2`).

### Step C — Enrich

Prepend each shard with a Global Context Header:
- Imported types and interfaces referenced in the shard
- Parent class signatures (if applicable)
- Relevant exported constants

Apply pruning before enriching:

| Strip | Retain (mandatory) |
|---|---|
| Inline comments | Type definitions |
| Dead code / unreachable blocks | Interface signatures |
| Console.log / debug prints | Exported constants and function signatures |
| Redundant whitespace | Imported type names and their origins |

### Step D — Self-check (orchestrator inline — no sub-skill)

Before sending each shard to code-auditor, the orchestrator answers three questions:

| Check | Pass condition | Fail action |
|---|---|---|
| Types resolved | All types/interfaces used in the shard are present in the Global Context Header | Add missing types from Core files; if unavailable, mark shard `Context Insufficient` and skip |
| Imports present | No unresolved import references remain after pruning | Re-enrich from Core files; if still unresolved, flag and skip |
| Shard is auditable | Shard contains real logic (not just blank lines / comments after pruning) | Mark `Empty After Pruning`, skip silently |

This is a fast internal check — not a sub-skill call. No architecture analysis here.
`system-auditor` is reserved for the synthesis stage (Step 3) when architecture context exists.

### Step E — Execute (→ code-auditor)

Read `/mnt/skills/user/code-auditor/SKILL.md`.
**Only pass shards that passed all three Step D checks.**
Directive: "Fix logic errors and style issues. Maintain all interface signatures and exported names."

### Step F — Track State

Maintain an `<audit-state>` block after each batch of max 5 shards.  
Format: `file:N:T:S` — N=shard count, T=C(ore)/L(eaf), S=✓(done)/⟳(fixing)/…(pending)/✗(fail)/⏸(halted)

```
<audit-state>types.ts:1:C:✓ | auth:3:C:⟳ | fmt:2:L:…</audit-state>
```

When a `HALT` signal is received mid-scan, freeze state and append checkpoint:
```
<audit-state>types.ts:1:C:✓ | auth:3:C:⏸[S2] | fmt:2:L:… | CHECKPOINT:FULL_REFACTOR</audit-state>
```
- `⏸[S2]` = halted at shard S2; resume from S2 if trigger is later canceled
- Trigger confirmed → discard checkpoint; pivot code-auditor to Technical Debt mode (see Interrupt Protocol)
- Trigger canceled → mark all `⏸` shards back to `⟳` and resume

After all shards are processed, continue to Steps 2b–4 for synthesis and implementation plan gate.

---

## Circuit Breaker Rules

| File Type | Failure Condition | Action |
|---|---|---|
| Any | Step D: `Context Insufficient` | Re-enrich from Core files; if still unresolved, skip shard and log in audit-state |
| Leaf | Step E: code-auditor fails 2× | Flag shard as `Logic Error`, skip, continue |
| Core | Step E: code-auditor fails any | **HALT** entire orchestration |

When halting on a Core failure:
```
⛔ ORCHESTRATION HALTED
Core file [FILE_ID] failed at Step E (code-auditor).
Reason: [brief reason]
Action required: [what the user needs to clarify or fix]
```

---

## Interrupt & Checkpoint Protocol

| Signal | Source | Orchestrator Action |
|---|---|---|
| `HALT: FULL_REFACTOR` | system-auditor | Freeze code-auditor at current `<audit-state>`. Mark pending shards `⏸`. Save checkpoint. Trigger plan gate immediately. |
| `HALT: CRITICAL_SEC` | either agent | Pause current batch. Show CRITICAL INTERRUPT gate. Await user before continuing. |

**Technical Debt mode** — directive sent to code-auditor when `FULL_REFACTOR` is confirmed:
> "Catalog only. List all issues with file, line, and tier. Do NOT generate fix recommendations. Maintain all interface signatures."

**Resume after canceled trigger:** restore checkpoint state, mark `⏸` → `⟳`, continue from saved shard ID.

---

## Constraint-Aware Routing

Check for user-supplied constraints BEFORE mode selection. Constraints override structural mode.

| Constraint phrase | Override |
|---|---|
| "freeze", "deploy in Xh", "no time for full audit" | Force **High-Severity Only** scope |
| "technical debt only" | Force code-auditor into **Technical Debt mode** (catalog, no fixes) |
| "security only" | Skip all non-security tiers and dimensions |
| "quick" / "just check X" | Already handled by Mode Priority 1 |

**High-Severity Only mode** — directive sent to code-auditor:
> "Report Critical and High tier issues only. Omit Medium and Low. Maintain all interface signatures."

When a constraint is applied, note it in the plan header:
```
Constraint applied: HIGH_SEVERITY_ONLY  Reason: production freeze (48h)
```

---

## Agent Prompt Templates

Prompts use wenyan-ultra to minimise tokens reproduced at each spawn.
Translation key for spawned agents unfamiliar with the encoding: 汝=You, 唯=only, 禁=forbidden, 域=scope, 限=max findings.

### code-auditor prompt (Pass 1)
```
汝：代碼審官。唯審實作，禁論架構拓撲。
域：設計｜正確｜安全｜測試。
碼：
<paste verbatim>
審畢。代碼：
- 問題：<issue>｜風：<risk>｜策：<fix>
無問題：<clean tiers>
限十。
發現零日/嚴重漏洞：立發 CRITICAL_INTERRUPT｜<issue>｜<class>。勿待審畢。
```

### system-auditor prompt (Pass 1)
```
汝：架構審官。唯審系統，禁論行級代碼。
域：整體設計｜資流｜技選｜擴性｜運維｜安全。
系：
<paste verbatim>
審畢。架構：
- 問題：<issue>｜風：<risk>｜策：<fix>
無問題：<clean dimensions>
限八。跨維衝突標：跨維：<dim>。
判需全重構：發 HALT: FULL_REFACTOR｜<reason>。判嚴重安全：發 HALT: CRITICAL_SEC｜<issue>。
```

### code-auditor cross-signal prompt (Pass 2)
```
汝：代碼審官。架構審官發現跨信：
[S-NN] <signal summary in wenyan-ultra>
問：此架構缺陷在代碼中有何具體暴露點？請標引用ID [S-NN]。
唯答暴露點，禁重複架構議題。限三。格式：[C-NN]←[S-NN]｜暴露：<location>｜風：<risk>
```

### system-auditor cross-signal prompt (Pass 2)
```
汝：架構審官。代碼審官發現跨信：
[C-NN] <signal summary in wenyan-ultra>
問：此代碼問題是否反映架構根因？若是，指明架構層何處。請標引用ID [C-NN]。
唯答架構根因，禁重複代碼症狀。限三。格式：[S-NN]←[C-NN]｜根因：<layer>｜策：<fix>
```

---

## Output Format (Final — English)

Issue IDs in the report MUST match those used in the Implementation Plan Gate.

```
ORCHESTRATION REPORT
════════════════════
Scope: <what was reviewed>
Mode: <⚡|S|D>
Agents used: <code-auditor | system-auditor | both>

── CODE AUDIT ──────────────────────────
[C-01] [Tier]: <description>
Solution: <precise fix>

── SYSTEM AUDIT ────────────────────────
[S-01] [Dimension]: <description>
Risk: <risk>
Recommendation: <actionable fix>

── CROSS-CUTTING ───────────────────────
[X-01] [C-NN] → [S-NN]: <code issue X is a symptom of system-level problem Y>
Fix: <unified resolution>

── CLEAN ───────────────────────────────
<tiers/dimensions with no issues>
```

Omit any section that is empty. Never show empty headers.
For **D** mode: final output MUST be passed through the `caveman` skill before presenting.

Append an execution graph at the end of every S and D report:
```
── EXECUTION GRAPH ─────────────────────
Mode: <S|D>  Constraints: <none|HIGH_SEV_ONLY|TECH_DEBT|SEC_ONLY>
Pass1:  code-auditor ∥ system-auditor
  signals: [C-NN]→sys · [S-NN]→code  (or: none)
Pass2:  code-auditor(cross) ∥ system-auditor(cross)  (or: skipped)
  interrupts: none | CRITICAL_INTERRUPT([C-NN]) | HALT_REFACTOR
Gates:  Plan Gate ·  CRITICAL gate([C-NN])  (or: Plan Gate only)
Result: proceed | canceled | scoped([C-NN,S-NN])
```

---

## Escalation Signals

After synthesis, flag these if present:

| Signal | Action | Agent Consequence |
|---|---|---|
| Code issue is symptom of architectural root cause | → Cross-Cutting section | none |
| system-auditor emits `HALT: FULL_REFACTOR` | → trigger plan gate immediately | halt code deep scan → Technical Debt mode |
| System has > 6 architectural issues | → standalone system-auditor deep dive | halt further code-auditor batches |
| Code has > 15 issues across tiers | → broader remediation plan; present to user before proceeding | pause; await user |
| Security issues at both code AND system level | → top of report | `HALT: CRITICAL_SEC` → CRITICAL INTERRUPT gate |

---

## Token Efficiency Notes

- Agents communicate in wenyan-ultra (~80–90% token reduction vs English)
- Orchestrator translates ONCE at output time
- No agent gets context it doesn't need
- Synthesis happens in orchestrator context, not in agent context
- If system-auditor would need > 8 findings, flag systemic issues only and point user to standalone system-auditor

---

## Startup Checklist

When this skill triggers:

1. Confirm sub-skill files are accessible:
   - `/mnt/skills/user/code-auditor/SKILL.md` (**D** Step E; **S** Step 2)
   - `/mnt/skills/user/system-auditor/SKILL.md` (**S**/**D** Step 3 synthesis only — **NOT** during shard validation or Step D self-check; confirm path here but do not call until synthesis)
   - `/mnt/skills/user/caveman/SKILL.md` (**D** final output)
2. Classify request → select mode (⚡ / S / D)
3. If **D**: ask user for file tree / code files if not provided
4. Begin `<wc>` analysis and output initial `<audit-state>` before first batch (**D** only)

---

## Reference

- code-auditor full behavior → `/mnt/skills/user/code-auditor/SKILL.md`
- system-auditor full behavior → `/mnt/skills/user/system-auditor/SKILL.md`
- wenyan-ultra compression → `/mnt/skills/user/caveman/SKILL.md`
- Never use wenyan-ultra for user-facing output unless user requests it
