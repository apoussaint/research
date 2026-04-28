# HANDOFF: AgentField-Style Orchestration Upgrade — Execution Pack

_Created: 2026-04-28 | For: Claude Code session running in `/Users/apous/.openclaw/workspace/apexcfo`_
_Source spec: `apexcfo/specs/SPEC_AGENTFIELD_STYLE_ORCHESTRATION.md`_
_Companion prompt: `apexcfo/prompts/claude-code-agentfield-orchestration.md`_

## Why this handoff exists

The review session that produced these recommendations was sandboxed to a
different workspace and could not edit the apexcfo source files directly. This
document is the complete, self-contained execution package: phased plan, schema
deltas vs. the original spec, validation steps, and non-goals. A Claude Code
session with write access to the apexcfo repo can execute this end to end
without re-reading the review thread.

## Decision summary

- **Execute the original spec** at `apexcfo/specs/SPEC_AGENTFIELD_STYLE_ORCHESTRATION.md`.
- Apply **two surgical schema tweaks** before writing any tool that consumes the
  schemas (cheap now, expensive to retrofit once consumers parse the v1 shape).
- Defer three additions (`RunScore`, `TelegramContext`, migration discipline
  doc) until evidence demands them — they are additive and safe to add later.
- Do **not** drift toward wholesale adoption: no planner, no merger agent, no
  one-call orchestration API, no replacement of the CLI/session/human-in-loop
  model.

## Phases

### Phase A — Schemas + routing config (target: ~1 hour, no behavior)

Create these files, content per original spec §4–§8 with the two tweaks below:

```
agents/orchestration/model-routing.json
agents/orchestration/task-schema.json
agents/orchestration/checkpoint-schema.json
agents/orchestration/debt-schema.json
```

#### Schema tweak 1 — split gates in `checkpoint-schema.json`

Original spec §7 has a flat `gates` object. Replace with:

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
- `model-routing.json`: §5 routing table
- `task-schema.json`: §5 classifier output shape
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
- All tools accept `--json` for machine-readable output.
- Exit non-zero on any schema violation.

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

All must succeed, write valid JSON, never require network.

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

## Done criteria

Original spec §11 verbatim, plus:

- `hard_gates` and `soft_gates` are separate keys in `checkpoint-schema.json`.
- `failures[]` entries include `recovery_action` (typed enum) and `chosen_by`.
- `docs/agent-orchestration.md` documents the hard-gate-fails-run rule.
- All §10 validation commands pass with no network access.

## Open question for the executing session

The original spec mentions `apexcfo/PROCESS_HEALTH.md` and `memory/ops-playbook.md`
as `read_first` references in the manifest example. Confirm those paths still
exist in the apexcfo repo before shipping the docs; update the example if any
have moved. Do not change the schema based on path drift — only the example.
