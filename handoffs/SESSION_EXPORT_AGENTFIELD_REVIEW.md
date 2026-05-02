# Session Export: AgentField Architecture Review and Adoption Plan

_Date: 2026-04-28_
_Branch: `claude/review-agentfield-architecture-9MRVo`_
_Repo: `apoussaint/research`_

---

## Session overview

A working session reviewing AgentField's "Beyond Vibe Coding" architecture,
comparing it to ApexCFO's existing agent orchestration, deciding what to
adopt vs. preserve, and producing an executable handoff for a separate
Claude Code session running in the apexcfo workspace.

Key outputs:
- `handoffs/AGENTFIELD_ORCHESTRATION_HANDOFF.md` — execution pack for
  apexcfo session (committed and pushed)
- This export — full conversation record

---

## 1. Initial request

**User:** Review AgentField's blog post and compare its architecture to ours.
Provided a DeepSeek summary of the comparison covering:

**Where AgentField is stronger:**
- Two clean primitives: `.ai()` (cheap structured calls) vs `.harness()`
  (autonomous coding loops)
- Typed failure recovery (inner retry, middle issue advisor, outer replanner)
- Checkpointing for resumable multi-agent builds
- Git worktree isolation per issue with intent-aware merging
- One-call orchestration API

**Where ApexCFO is stronger:**
- Financial correctness gates (parity, DSO/ARR, RLS, PE auditability)
- Process integrity culture (PROCESS_HEALTH, Red Team, circuit breakers,
  Section 6b, handoffs, memory stewardship)
- Human business context (Alan/project memory, customer specifics)
- Practical production deployment (Render/Vercel/Supabase/Stripe/Intuit)

**Biggest gap:** Too much architecture lives as ritual instead of machinery.

**What to steal:** `.ai()` equivalent, `.harness()` equivalent, typed debt,
checkpointed builds, worktree-per-agent, model routing map.

---

## 2. Initial plan (later revised)

Proposed seven-phase plan adopting all six AgentField patterns:
- Phase 0: Foundation (contracts, schemas, model-routing config)
- Phase 1: `.ai()` primitive
- Phase 2: `.harness()` primitive
- Phase 3: Typed debt + 3-loop recovery
- Phase 4: Checkpointed builds
- Phase 5: Worktree-per-agent
- Phase 6: One-call orchestration

This drifted toward wholesale adoption. Later revised after user pushback.

---

## 3. Impact analysis: scoring, learning loop, promotion logic

User asked: will adoption impact our scoring and learning loop and promotion
logic?

**Scoring (Red Team):** RT becomes a `harness()` exit code, mechanically
un-bypassable. Risk: flows that quietly passed on human nod will now fail.
Mitigation: Phase 2a — RT threshold calibration window before enforcement.

**Learning loop:** Structured TypedDebt + Checkpoints + advisor decisions
replace prose handoffs. Net improvement (queryable, replayable). Risk:
downstream prose parsers break at cutover. Mitigation: Phase 3a — dual-write
prose + TypedDebt for one cycle.

**Promotion logic:** Becomes evidence-based. `harness().score` + TypedDebt
frequency per (role, model) tuple lets you promote on data, not vibes.
Risk: Phase 1 risk classifier becomes single point of failure for routing.
Mitigation: Phase 1a — shadow-mode classifier for two weeks before gating.

**Recommended sequencing:** insert calibration "a" phases as guard rails:
0 → 1 → 1a → 2 → 2a → 3 → 3a → 4 → 5 → 6.

---

## 4. Scoring logic deep dive

User asked specifically about scoring logic.

Today: `score = RedTeam.score`, separate boolean gates, human composes them.

**Proposed `RunScore` object:**
```
RunScore = {
  hard_gates: {                    # 0/1, multiplicative
    financial_parity: bool,
    rls_auth: bool,
    pe_auditability: bool,
    red_team_floor: bool,
    tests_pass: bool,
  },
  soft_score: float in [0,1],
  components: {
    red_team, test_coverage_delta, debt_penalty, review_signal
  },
  confidence: { n_runs, variance },
  attribution: { role, model, run_id, commit_sha, checkpoint_id }
}
```
`final_pass = all(hard_gates) AND soft_score >= role_threshold`.

**Three logic changes that matter:**

1. **Hard vs soft separation.** Domain gates (parity, RLS, auditability)
   are multiplicative 0/1. A run brilliant on RT but breaking RLS is a fail.
2. **Debt-aware penalty.** `debt_penalty = sum(severity_weight) / role_debt_budget`.
   Closes the loophole where agents pass by deferring everything.
3. **Per-role normalization.** Cron's RT-9 ≠ coder's RT-9. Role-specific
   weights and thresholds in `MODEL_ROUTING.yaml`.

**Promotion logic:**
```
promote(model: M, from: role_A, to: role_B) iff
  E[soft_score | M, role_A] >= threshold(role_B) + safety_margin
  AND P(hard_gate_fail | M, role_A) <= role_B.tolerated_fail_rate
  AND n_runs >= role_B.min_evidence
```

**Risks:** Goodhart on soft_score, score drift across model versions,
hard-gate brittleness, aggregating across run scopes.

---

## 5. AgentField test run

User asked whether to run AgentField's SWE-AF planner first. Recommended yes,
on a small throwaway target, to harvest concrete schema shapes and avoid
designing schemas from blog summaries.

**Recommended test goal:**
> Build a CLI tool `ledger` in Python that reads a CSV of dated financial
> transactions (header: date,account,amount,description), enforces two-decimal
> banker's rounding, computes a running balance per account, detects duplicate
> transactions (same date+amount+description within 24h), and supports
> `--format json|yaml|table`. Exit non-zero on parse errors. Include a pytest
> suite covering rounding edge cases, duplicate detection, and malformed input.

Why: ~6–8 issues, self-contained (no DB/auth/network), deterministic, in
user's domain so they can spot bullshit, ~$10–15.

**Walkthrough of what happens:**
1. Planner agent (`.ai()`) produces issue graph with dependencies
2. Async response with job ID; build runs in background
3. Worktree fan-out per issue
4. Harness agents run in parallel within each worktree
5. Issue advisor handles failures via typed actions
6. Checkpoints between dependency levels
7. Merger agent integrates branches in dependency order
8. Final: draft PR + per-issue scores + debt log + cost

Expected duration: 20–60 min.

---

## 6. Telegram interface impact

User asked: will the Telegram interface still work?

Yes — it's a transport layer, not part of the refactor. Four integration
points to handle:

1. **Handler becomes a thin shim**, not an orchestrator. Calls
   `ai(msg, IntentSchema)` then `harness_async(intent.goal, ...)`.
2. **Latency model changes.** harness() runs 20–60 min; Telegram users expect
   ack in seconds. Add progress pings.
3. **Group-chat boundaries plumbed through `harness().context.actor`.**
   Highest-risk integration point. Boundary moves from handler to typed
   request context. Must enforce as hard gate.
4. **Reply rendering.** Need structured-to-prose layer
   (`render_for_telegram(RunScore)`).

**Cutover risk:** Phase 2 → 2a transition. RT flips from advisory to hard-fail.
Users will see "rejected" replies they didn't see before. Mitigation: render
rejections with reason + override affordance.

**Phase 0 addition:** `TelegramContext` schema with `actor`, `chat_id`,
`chat_kind`, `pii_visibility`, `allowed_repos`, `allowed_models`. Required as
hard gate input to harness.

---

## 7. Wholesale vs surgical pushback

User pushed back: "I kind of like the idea of stealing certain elements like
DeepSeek suggested." Acknowledged the original plan drifted toward wholesale,
especially Phase 6.

**Re-cut as surgical matrix:**

| AgentField pattern | Verdict |
|---|---|
| TypedDebt schema | Steal verbatim |
| Worktree-per-issue | Steal verbatim |
| Model routing as YAML | Steal pattern |
| Checkpoint shape | Steal verbatim |
| `.ai()` primitive | Steal pattern, not abstraction |
| `.harness()` primitive | Steal for ONE entry point first |
| 3-loop typed recovery | Steal only middle loop (issue_advisor) |
| Planner → issue graph | Adopt for low-risk only |
| Merger agent | Don't steal |
| One-call orchestration API | Don't steal |

**Revised plan:** Phase 0 contracts → Phase 1 quick wins → Phase 2 `.ai()`
discipline → Phase 3 `.harness()` for one flow → Phase 4 issue advisor
(optional) → Phase 5 checkpointing (only if needed). Phase 6 deleted.

**Net effect:** ~4 sprints additive instead of 7+ refactor; moat
(domain gates + human-in-loop ops) preserved.

---

## 8. Comparison with apexcfo's existing spec

User shared `apexcfo/specs/SPEC_AGENTFIELD_STYLE_ORCHESTRATION.md`.

**Spec primitives:**
1. Task classifier (rule-based first, no LLM in v1)
2. Harness manifest (pre-execution contract, not runtime wrapper)
3. Checkpoint records
4. Typed debt records
5. Model routing policy
+ Worktree policy (docs only, not automated)

**Alignment scorecard:**

| Pattern | My matrix | Their spec | Verdict |
|---|---|---|---|
| TypedDebt | Steal verbatim | §8 full schema + CLI | Match |
| Worktree | Steal verbatim | §9 docs-only | More conservative — defensible |
| Model routing | Steal pattern | §5 JSON, rule-based | Match |
| Checkpoint | Steal verbatim | §7 full schema + CLI | Match |
| `.ai()` | Steal pattern, thin wrapper | §5 rule-based, no LLM v1 | Smarter |
| `.harness()` | Wrap one flow | §6 manifest only | Different but cleaner |
| Issue advisor | Optional Phase 4 | Not present | Missing |
| Planner | Low-risk only | Not present | Correctly dropped |
| Merger | Don't steal | Not present | Correctly dropped |
| One-call API | Don't steal | Not present | Correctly dropped |

7/10 patterns match, 2 correctly dropped, 1 missing piece worth adding.

**Where the spec is better than my plan:**
- Manifest-vs-wrapper split is cleaner than my conflation
- Rule-based classifier first defers LLM dependency
- No-network constraint eliminates flakiness
- Concrete file paths (mine handwaved)
- Strong non-goals section locks down moat
- Sized realistically ("1–2 Claude Code sessions")

**Missing additions worth considering:**
1. `RunScore` / composite scoring — defer (can be additive later)
2. Hard-gate vs soft-score separation — bake in NOW
3. `TelegramContext` schema — defer (additive)
4. Calibration phases (dual-write, shadow-mode, RT calibration) — defer to docs
5. Typed `recovery_action` in failures — bake in NOW

---

## 9. Final recommendation: original spec + 2 tweaks

User asked: original spec or revised?

**Recommendation: ship the original verbatim with two surgical tweaks.**

Cheap-now/expensive-later test:

| Tweak | Cost now | Cost to retrofit |
|---|---|---|
| Hard/soft gate split | 5 lines | High (consumers parse flat shape) |
| Typed `recovery_action` | 1 enum | High (parsers infer from prose) |
| `RunScore` | New file + tooling | Low (separate file) |
| `TelegramContext` | New field | Low (additive) |
| Migration discipline | New doc | Low (pure docs) |

First two: bake in. Other three: defer.

**Tweak 1 — split gates in `checkpoint-schema.json`:**
```json
"hard_gates": {
  "financial_parity": "PASS|FAIL|N/A",
  "rls_auth":         "PASS|FAIL|N/A",
  "pe_audit":         "PASS|FAIL|N/A"
},
"soft_gates": {
  "frontend_critic":  "PASS|FAIL|SKIP",
  "backend_critic":   "PASS|FAIL|SKIP",
  "red_team":         "PASS|FAIL|SKIP"
}
```
Rule: any `hard_gates` failure fails the run regardless of `soft_gates`.

**Tweak 2 — typed recovery action in `failures[]`:**
```json
{
  "phase": "...",
  "summary": "...",
  "resolution": "...",
  "recovery_action": "retry|narrow_scope|defer_with_debt|ask_human|replan|none",
  "chosen_by": "human|advisor"
}
```

---

## 10. JSON vs YAML for routing config

User asked: JSON or YAML for the routing doc?

| | JSON | YAML |
|---|---|---|
| Comments | none | yes |
| Hand-editing | verbose | cleaner |
| Parse footguns | none | Norway problem, octal, indent |
| Stdlib in Python | built-in | needs pyyaml |
| Schema validation | mature | weaker |
| Cross-runtime | identical | parsers disagree |

**Recommendation: JSON, with rationale in a sibling markdown file.**
- Spec already commits to JSON
- Zero new deps
- Norway problem alone is reason to avoid YAML for routing config that gates
  which model touches financial code
- If comments become essential, JSON5/JSONC is a smaller migration than YAML

Pattern:
```
agents/orchestration/model-routing.json       # data
agents/orchestration/model-routing.NOTES.md   # rationale
```

---

## 11. Cross-repo scope (global framework)

User raised: framework should apply globally to claudeia, openclaw, and
apexcfo (which already share documentation).

**Insight:** Don't redesign as shared platform from day 1. That's the
wholesale trap. Ship in apexcfo first, but bake five extensibility decisions
into v1 schemas so v2 extraction is non-breaking.

**Five extensibility decisions (mandatory in v1):**

1. **`$schema_version`** on every schema file (semver, tools fail loudly on
   mismatch)
2. **`hard_gates` is a typed map**, not fixed enum — apexcfo registers
   `financial_parity`, `rls_auth`, `pe_audit` as values; claudeia/openclaw
   register their own
3. **`model-routing.json` reserves `extends` field** — `null` in v1, supports
   layering in v2
4. **`task_type` is open string**, not closed enum — apexcfo-known values
   live in docs, not schema
5. **Per-repo config file `.agent-orchestration.json`** at repo root, declaring
   `repo_id`, `hard_gates`, `default_read_first`, `default_forbidden_scopes`

**New phases (future, not in v1):**
- Phase D: Extract schemas + tools to canonical core location
- Phase E: Per-repo onboarding (each repo writes its `.agent-orchestration.json`)

**Strategic decision: canonical core location.**
Recommended openclaw, because:
- Already the orchestration platform
- Already shares documentation with claudeia
- Can carve out separate repo later if growth demands; carving back in is harder

**v1 effort unchanged at 1–2 sessions.** The five decisions cost ~30 min of
design and zero implementation effort.

---

## 12. Locked decision: openclaw as canonical core

User confirmed openclaw as the v2 home for shared schemas + tools at
`openclaw/agent-orchestration/`, consumed by apexcfo and claudeia via git
submodule or editable pip install. v1 paths in apexcfo remain provisional;
Phase D moves them.

---

## 13. Verification request

User asked: check what was built against the plan.

**Result:** Cannot verify. Accessible only to:
- `apoussaint/research` (this repo) — only handoff commits, no execution
  artifacts
- Local sandbox `/home/user/research` — same

Not accessible:
- `/Users/apous/.openclaw/workspace/apexcfo/` (user's Mac)
- apexcfo GitHub repo (not in session allowlist)

**To verify, user must provide one of:**
1. Diff (`git diff` output from apexcfo)
2. Push artifacts to a branch in `apoussaint/research`
3. Paste file listing + schema contents

**Verification checklist provided** covering Phase A schemas, Phase B tools,
Phase C docs, validation, and non-goals.

---

## 14. Artifacts produced

Committed to `apoussaint/research`, branch
`claude/review-agentfield-architecture-9MRVo`:

1. `handoffs/AGENTFIELD_ORCHESTRATION_HANDOFF.md` (commits 24442d0, 73c60f9, 883ee02)
   - Phased execution pack for apexcfo session
   - Two surgical schema tweaks
   - Five cross-repo extensibility decisions
   - Phases D and E reserved for future
   - openclaw confirmed as canonical core home

2. This export.

---

## 15. Decision log (one-liners)

- **Adopt:** TypedDebt verbatim, worktree (docs-only), checkpoint, model
  routing config, manifest-as-contract, rule-based classifier
- **Adopt with tweaks:** hard/soft gate split in checkpoint, typed
  `recovery_action` enum in failures
- **Defer:** RunScore composite scoring, TelegramContext schema, migration
  discipline doc, automated worktree tooling, issue advisor (middle recovery
  loop)
- **Reject:** planner agent, merger agent, one-call orchestration API,
  replacement of CLI/session/human-in-loop model
- **Locked:** openclaw as canonical v2 core home; apexcfo ships v1 first;
  claudeia and openclaw onboard in Phase E
- **Schema config format:** JSON (with sibling NOTES.md for rationale)
- **v1 sizing:** 1–2 Claude Code sessions, unchanged after cross-repo additions

---

## 16. Open items

1. Verify what was built in apexcfo against the handoff checklist (waiting on
   user to share artifacts)
2. Confirm `read_first` paths in apexcfo manifest example still exist
3. Confirm apexcfo hard gate names (`financial_parity`, `rls_auth`,
   `pe_audit`) match existing process docs; rename if drift exists
4. After v1 runs in apexcfo for ≥2 weeks, plan Phase D extraction to openclaw
5. After Phase D, onboard claudeia and openclaw in Phase E (~30 min each)
