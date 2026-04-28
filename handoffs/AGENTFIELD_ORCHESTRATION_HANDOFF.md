# HANDOFF: AgentField-Style Orchestration Upgrade — Execution Pack

_Created: 2026-04-28 | Updated: 2026-04-28 (cross-repo extensibility)_
_For: Claude Code session running in `/Users/apous/.openclaw/workspace/apexcfo`_
_Source spec: `apexcfo/specs/SPEC_AGENTFIELD_STYLE_ORCHESTRATION.md`_
_Companion prompt: `apexcfo/prompts/claude-code-agentfield-orchestration.md`_

## Why this handoff exists

The review session that produced these recommendations was sandboxed to a
different workspace and could not edit the apexcfo source files directly. This
document is the complete, self-contained execution package: phased plan, schema
deltas vs. the original spec, validation steps, non-goals, and cross-repo
extensibility decisions. A Claude Code session with write access to the
apexcfo repo can execute Phases A–C end to end without re-reading the review
thread.

## Decision summary

- **Execute the original spec** at `apexcfo/specs/SPEC_AGENTFIELD_STYLE_ORCHESTRATION.md`.
- Apply **two surgical schema tweaks** before writing any tool that consumes the
  schemas (cheap now, expensive to retrofit once consumers parse the v1 shape).
- Apply **five cross-repo extensibility decisions** in v1 schemas so the
  primitives can be extracted to a shared core later without breaking
  consumers in claudeia and openclaw.
- Defer three additions (`RunScore`, `TelegramContext`, migration discipline
  doc) until evidence demands them — they are additive and safe to add later.
- Do **not** drift toward wholesale adoption: no planner, no merger agent, no
  one-call orchestration API, no replacement of the CLI/session/human-in-loop
  model.

## Cross-repo scope (READ FIRST)

The orchestration primitives are intended to be **global** across at least
three workspaces that already share documentation:

- `apexcfo` — financial software, strict domain gates
- `claudeia`
- `openclaw` — orchestration platform

**v1 ships in apexcfo only.** The schemas and tools are written into apexcfo
paths to keep v1 small and concrete. But because claudeia and openclaw will
consume the same primitives in v2, **five extensibility decisions must be
made in v1** so v2 extraction is non-breaking. These are mandatory in Phase A.

### Decision needed before Phase A starts

**Where will the shared core live in v2?** Recommended: a new
`agent-orchestration/` subdirectory inside the **openclaw** repo, consumed
by apexcfo and claudeia via git submodule or a small editable pip install.
Reasons:
- openclaw is already the orchestration platform; the schemas conceptually
  belong there
- claudeia and openclaw already share documentation, so the path pattern exists
- a separate repo can be carved out later if growth justifies it; carving back
  in is harder

This decision affects **only naming and prose** in v1; no v1 paths change.
Confirm or override before Phase A begins.

### The five extensibility decisions (apply in Phase A)

These are mandatory in v1 schemas. They cost ~30 minutes of design and zero
extra implementation effort. Skipping any of them creates a v2 migration on
three repos.

**1. `$schema_version` on every schema file.**
Add `"$schema_version": "0.1.0"` (semver) at the top of every
`agents/orchestration/*-schema.json`. Tools must check this field on load and
fail loudly on mismatch.

**2. `hard_gates` is a typed map, not a fixed enum.**
The schema declares `hard_gates` as an object whose keys are arbitrary strings
and values are `"PASS|FAIL|N/A"`. The apexcfo v1 instance registers
`financial_parity`, `rls_auth`, `pe_audit` as values inside that map.
claudeia and openclaw will register different gate names without touching the
shared schema.

**3. `model-routing.json` reserves an `extends` field.**
Schema permits an optional `"extends": "<path-or-null>"` at the top level. v1
in apexcfo writes `"extends": null`. v2 will support layering: a base config
at the canonical openclaw location plus per-repo overrides. The field must be
reserved now or v2 consumers will need a migration.

**4. `task_type` is an open string, not a closed enum.**
Original spec §5 lists `bugfix|feature|ui|backend|finance|auth|deploy|cron|docs|research`.
In the schema, drop the closed enum; declare `task_type` as `string`. Document
the apexcfo-known values in the docs and in the rule-based classifier, but do
not bake them into the schema. claudeia/openclaw will have different task
types (e.g., `prompt_eng`, `harness_build`, `eval`).

**5. Per-repo config file `.agent-orchestration.json` at repo root.**
Each repo declares its own:
```json
{
  "$schema_version": "0.1.0",
  "repo_id": "apexcfo",
  "hard_gates": ["financial_parity", "rls_auth", "pe_audit"],
  "default_read_first": ["AGENTS.md", "memory/ops-playbook.md", "PROCESS_HEALTH.md"],
  "default_forbidden_scopes": []
}
```
Tools read this on startup and use it to populate manifests, classify scopes,
and enforce hard gates. apexcfo writes its file in v1. claudeia and openclaw
drop in their own when they adopt.

## Phases

### Phase A — Schemas + routing config (target: ~1 hour, no behavior)

Create these files, content per original spec §4–§8 with the two surgical
tweaks **and** the five extensibility decisions applied:

```
agents/orchestration/model-routing.json     # has "extends": null
agents/orchestration/task-schema.json       # task_type: string (open)
agents/orchestration/checkpoint-schema.json # hard_gates as typed map; soft_gates separate
agents/orchestration/debt-schema.json
.agent-orchestration.json                   # repo root
```

Every schema file gets `"$schema_version": "0.1.0"` at the top.

#### Schema tweak 1 — split gates in `checkpoint-schema.json`

Original spec §7 has a flat `gates` object. Replace with:

```json
"hard_gates":  { "<gate_name>": "PASS|FAIL|N/A" },
"soft_gates":  { "<gate_name>": "PASS|FAIL|SKIP" }
```

The set of valid `<gate_name>` keys for `hard_gates` is determined by the
repo's `.agent-orchestration.json` `hard_gates` array. Tools must validate
that all declared hard gates are present in any checkpoint they write.

apexcfo's instance therefore writes:
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

Rule documented in `docs/agent-orchestration.md`:
> Any `hard_gates` value of `FAIL` fails the run regardless of `soft_gates`
> outcomes. `hard_gates` are multiplicative; `soft_gates` are advisory plus a
> per-role threshold.

#### Schema tweak 2 — typed recovery action in `failures[]`

Original §7 `failures[]` entries are `{phase, summary, resolution}`. Add:

```json
{
  "phase": "...",
  "summary": "...",
  "resolution": "...",
  "recovery_action": "retry|narrow_scope|defer_with_debt|ask_human|replan|none",
  "chosen_by": "human|advisor"
}
```

Both fields are required; `chosen_by: "human"` is the v1 default. The
`advisor` value is reserved for a future automated middle-loop and does not
need to be implemented now — only the schema slot.

#### Everything else verbatim from original spec
- `model-routing.json`: §5 routing table (plus `"extends": null`)
- `task-schema.json`: §5 classifier output shape (with `task_type` as open string)
- `debt-schema.json`: §8 typed debt records
- Manifest fields: §6

### Phase B — CLI tools (target: ~2–3 hours)

Create:

```
tools/agent_route.py        # classify + manifest subcommands
tools/agent_checkpoint.py   # init / update / show
tools/agent_debt.py         # add / list / close
```

Constraints:
- **No network calls.** Tools must run fully offline.
- Classifier (`agent_route.py classify`) is **rule-based** in v1 — keyword and
  path-pattern heuristics over the task text. No LLM call.
- All tools read schemas from `agents/orchestration/*.json` and validate
  inputs/outputs against them.
- All tools read `.agent-orchestration.json` at repo root and use its
  `hard_gates`, `default_read_first`, `default_forbidden_scopes`,
  `repo_id` to populate manifests and validate checkpoints.
- All tools check `$schema_version` on load and fail loudly on mismatch.
- All tools accept `--json` for machine-readable output.
- Exit non-zero on any schema violation.
- **No apexcfo-hardcoded paths in tool code.** All repo-specific values must
  come from `.agent-orchestration.json`. This is a hard requirement: it is
  what makes Phase D extraction non-breaking.

CLI surface: see original spec §5, §6, §7, §8 verbatim.

### Phase C — Docs + validation (target: ~1 hour)

Create `docs/agent-orchestration.md` with:

1. **Mapping table:** AgentField primitive → ApexCFO primitive → existing
   process gate (Red Team, Section 6b, PROCESS_HEALTH, handoffs, memory).
2. **Hard-gate rule:** any `hard_gates` failure fails the run.
3. **Worktree policy:** verbatim from original spec §9 (docs-only, not
   automated in v1).
4. **How this maps to current ritual:** explicit short paragraphs naming
   `AGENTS.md`, `SOUL.md`, `USER.md`, handoffs, memory, Red Team, Section 6b,
   so the new primitives are recognizable to operators using the existing
   bootstrap.
5. **Cross-repo note:** the schemas and tools will move to a canonical
   location (recommended: `openclaw/agent-orchestration/`) in Phase D.
   v1 paths in apexcfo are provisional; per-repo behavior is configured via
   `.agent-orchestration.json` and is already extraction-ready.

Run validation per original spec §10:

```bash
python tools/agent_route.py classify --task "Move low-risk crons to GLM 5.1" --json
python tools/agent_route.py manifest --task "Fix pricing checkout redirect" --id test-pricing-checkout --json
python tools/agent_checkpoint.py init --task-id test-checkpoint --status planned
python tools/agent_checkpoint.py update --task-id test-checkpoint --status done --gate frontend_critic=PASS
python tools/agent_checkpoint.py show --task-id test-checkpoint
python tools/agent_debt.py add --id test-debt --type process --severity low --summary "Test debt"
python tools/agent_debt.py list --open
```

All must succeed, write valid JSON, never require network. Add one extra
validation:

```bash
# Confirm tools fail loudly on schema version mismatch
python -c "import json; d = json.load(open('agents/orchestration/checkpoint-schema.json')); d['\$schema_version'] = '99.0.0'; json.dump(d, open('/tmp/bad-schema.json','w'))"
# Pointing a tool at /tmp/bad-schema.json must exit non-zero.
```

### Phase D — Extract to shared core (FUTURE, NOT IN V1)

After v1 has run in apexcfo for long enough to validate the primitives
(suggest ≥2 weeks of real use), extract:

```
agents/orchestration/*.json   →  <canonical>/agent-orchestration/schemas/
tools/agent_*.py              →  <canonical>/agent-orchestration/tools/
```

Where `<canonical>` is the location decided up front (recommended: openclaw).
Apexcfo, claudeia, openclaw consume via submodule or `pip install -e`.
Update `.agent-orchestration.json` files in each repo if needed. No schema
changes should be required — that is the point of Phase A's extensibility
decisions.

### Phase E — Per-repo onboarding (FUTURE, NOT IN V1)

Each new repo writes its own `.agent-orchestration.json`:
- claudeia: declares its own hard gates (likely none or `prompt_safety`),
  read_first list, forbidden_scopes
- openclaw: declares its own hard gates (likely `runtime_safety`,
  `agent_isolation`), read_first list

Estimated ~30 min per repo.

## Non-goals — do not relax (verbatim from original spec §3)

- Do **not** build a new agent runtime.
- Do **not** rewrite Antfarm.
- Do **not** change product UI or ApexCFO customer-facing flows.
- Do **not** alter billing, QBO, financial math, auth, RLS, or deployment logic.
- Do **not** auto-send Telegram/email alerts from new code without explicit
  existing patterns and dry-run support.
- Do **not** enable disabled crons or create new live scheduled jobs.

Plus from this review:
- Do **not** add `RunScore` composite scoring in v1 — defer until promotion
  logic needs it.
- Do **not** add `TelegramContext` schema in v1 — additive field, safe to add
  later.
- Do **not** add a planner agent, merger agent, or one-call orchestration API.
  These are wholesale moves outside the surgical scope.
- Do **not** attempt Phase D or Phase E in this session. Extraction happens
  only after v1 has accumulated real evidence in apexcfo.
- Do **not** create files in claudeia or openclaw repos in this session.
  v1 ships in apexcfo only.

## Done criteria

Original spec §11 verbatim, plus:

- `hard_gates` and `soft_gates` are separate keys in `checkpoint-schema.json`,
  with `hard_gates` declared as a typed map (open key set).
- `failures[]` entries include `recovery_action` (typed enum) and `chosen_by`.
- Every `agents/orchestration/*.json` carries `$schema_version: "0.1.0"`.
- `model-routing.json` includes `"extends": null` reserved for v2 layering.
- `task_type` in `task-schema.json` is `string`, not a closed enum.
- `.agent-orchestration.json` exists at apexcfo repo root with `repo_id`,
  `hard_gates`, `default_read_first`, `default_forbidden_scopes`.
- Tools read `.agent-orchestration.json` and contain no apexcfo-hardcoded
  paths.
- `docs/agent-orchestration.md` documents the hard-gate-fails-run rule and
  notes Phase D extraction intent.
- All §10 validation commands pass with no network access.
- Schema-version mismatch causes tools to exit non-zero.

## Open questions for the executing session

1. **Canonical core location.** Confirm openclaw is the right home for the
   shared core in v2, or specify an alternative. This is a naming/prose
   decision in v1 only — does not block Phase A.
2. **`read_first` paths.** The original spec mentions `apexcfo/PROCESS_HEALTH.md`
   and `memory/ops-playbook.md` in the manifest example. Confirm those paths
   still exist; update the example if any have moved. Do not change schemas
   based on path drift.
3. **Hard gate names.** Confirm the apexcfo v1 hard gates are
   `financial_parity`, `rls_auth`, `pe_audit`. If a gate is named differently
   in existing process docs, prefer the existing name.
