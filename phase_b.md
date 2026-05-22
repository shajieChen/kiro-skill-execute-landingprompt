# Execute-LandingPrompt — Phase B 回流 (companion)

Loaded on-demand by ELP SKILL.md Phase B (Steps 10-13). Contains the approved_transitions.json schema, scaffold rules, data mapping, HC management, direct-write meta refresh, idempotency, change_event format, and sequential ID generation algorithm.

Error Handling stays in SKILL.md as `§ Phase B Error Handling` — it MUST be consulted on every Phase B failure regardless of whether this companion was loaded.

## § PST 回流

### Trigger

After Phase A completes (regardless of completed/partial/blocked), immediately execute Phase B.

### Write Method

Priority order:
1. If `tools/apply_changes.py` exists in pst_root → write `status/.cache/approved_transitions.json` then invoke pipeline.
2. If tools/ does not exist → direct-write status.yaml (then refresh meta block inline).

**Scaffold phase exception:** When `status/status.yaml` does not yet exist, ELP MUST use direct-write regardless of whether `tools/apply_changes.py` exists. `apply_changes.py` requires an existing status.yaml (it exits with code 2 if missing) and cannot bootstrap. After scaffold, subsequent ELP runs follow the normal priority order above.

### § approved_transitions.json Format (for apply_changes.py path)

When writing through `apply_changes.py`, ELP writes the following JSON to `<pst_root>/status/.cache/approved_transitions.json`:

```json
{
  "event_summary": "ELP: <one-line summary of execution result>",
  "event_type": "lp_execution",
  "transitions": [
    {
      "artifact": "<resolved LP artifact id>",
      "type": "landing_prompt",
      "from": "<previous_status or null>",
      "to": "<ready|needs_update|blocked>",
      "reason": "ELP: <one-line summary>",
      "source": "Execute-LandingPrompt"
    }
  ]
}
```

**Critical:** Every transition object MUST include `"source": "Execute-LandingPrompt"`. This field is how PST identifies ELP-authored transitions for auto-approval during AUDIT Step 4.

If HC needs creation, append a handoff operation to the transitions array:

```json
{
  "op": "handoff_register",
  "handoff_id": "HC-<next>",
  "producer": "<artifact id>",
  "producer_type": "landing_prompt",
  "to": "available",
  "version": 1,
  "title": "<LP title or filename>",
  "facts": [
    {"id": "F-001", "statement": "<confirmed item>", "source": "<lp_file_path>", "confidence": "high"}
  ],
  "constraints": [
    {"id": "C-001", "statement": "<unresolved item>", "source": "<lp_file_path>"}
  ],
  "consumed_by": ["<next LP id>"],
  "reason": "ELP execution produced handoff context",
  "source": "Execute-LandingPrompt"
}
```

Or for updating an existing HC (version bump — MUST carry the new content; bumping version without content is forbidden):

```json
{
  "op": "handoff_version",
  "handoff_id": "HC-<existing>",
  "version": "<new version number>",
  "to": "available",
  "facts": [
    {"id": "F-001", "statement": "<updated confirmed item>", "source": "<lp_file_path>", "confidence": "high"}
  ],
  "constraints": [
    {"id": "C-001", "statement": "<updated unresolved item>", "source": "<lp_file_path>"}
  ],
  "reason": "ELP re-execution: facts/constraints changed",
  "source": "Execute-LandingPrompt"
}
```

**Required:** `facts[]` and `constraints[]` MUST be present in every `handoff_version` op and MUST reflect the post-execution state in full (replacement semantics, not diff). `apply_changes.py` replaces the HC's facts/constraints arrays with the supplied content and only then increments `version`. A `handoff_version` op missing these arrays is rejected by ELP's pre-write self-check (do not emit the JSON). Empty arrays (`facts: []`, `constraints: []`) satisfy the presence requirement — only a completely absent key is rejected.

> **Op key convention:** New ELP writes MUST use `handoff_id`. `apply_changes.py` still accepts the legacy `artifact:` key for backward compatibility, but ELP MUST NOT emit it.

After writing the JSON, invoke: `python tools/apply_changes.py --project <pst_root>`

### Scaffold Rules

If `<pst_root>/status/status.yaml` does not exist, create minimal structure:

```
<pst_root>/
├── status/
│   ├── status.yaml
│   └── .cache/
├── prompts/
│   ├── landing/
│   └── test/
├── Result/
└── (views/ and tools/ are NOT created — PST INIT handles those)
```

**Post-scaffold note:** After ELP scaffolds a minimal status.yaml, the user should run PST INIT (`Skill project-state-tracker + init`) to generate `tools/` and `views/` directories. Until then, ELP will use direct-write mode for subsequent executions.

Minimal status.yaml — populate per PST §6G schema, with these scaffold-specific values:

- `meta.project_name` / `project.name` = pst_root folder name
- `meta.schema_version` = 1, `project.phase` = "execution", `project.version` = "1.0"
- `meta.{created,last_updated,last_run}` = current ISO timestamp
- `meta.{source_root,scope,pst_root,coding_standards}` = resolved README values (omit any key whose source is absent or a placeholder — do NOT write `"<未配置>"` into meta)
- `meta.summary` = recounted values (typically `artifacts_total: 1`, `handoffs_total: 1`, `handoffs_pending_consumers: 1`, others 0)
- `meta.pointers` = standard defaults (`entry_point: "AGENTS.md"`, `views_dir: "views/"`, `status_report: "views/status_report.md"`, `handoff_view: "views/handoff_view.md"`, `prompt_chain: "views/prompt_chain_view.md"`)
- `artifacts`: single record for the current LP — `{id: <resolved>, type: landing_prompt, path: <resolved>, status: <ready|needs_update|blocked>, depends_on: [], produces_handoffs: [HC-001], consumes_handoffs: [], last_checked: <ISO>}`
- `rules` = all five flags `true` (research_is_fact_only, status_is_single_source_of_truth, landing_requires_test_ready, plan_invalidates_landing, landing_invalidates_test)
- `handoff_contexts`: single `HC-001` for the current LP per § Data Mapping (structured facts/constraints, `status: available`, `version: 1`, `consumed_by: [<next LP id>]`, `consumed_status: [{consumer: <next LP id>, status: pending, consumed_version: null, consumed_at: null}]`)
- `snapshots`: `{enabled: true, git_baseline: null, file_hashes: {}}`
- `change_events`: single `CE-001` with `source: Execute-LandingPrompt`, `event_type: scaffold_and_execute`, one transition `{from: null, to: <ready|needs_update|blocked>}`
- Empty arrays for: `research_findings`, `evidence`, `assumptions`, `decisions`, `dependencies`, `blockers`, `gates`, `preconditions`

### Data Mapping

| ELP Output | PST Field | Rule |
|----------|----------|------|
| LP filename | `artifacts[].id` | Per § ID Resolution |
| completed | `artifacts[].status` | → `ready` |
| partial | `artifacts[].status` | → `needs_update` |
| blocked | `artifacts[].status` | → `blocked` |
| Each confirmed item | `handoff_contexts[].facts[]` | Structured: `{id, statement, source, confidence}` |
| Each unresolved item | `handoff_contexts[].constraints[]` | Structured: `{id, statement, source}` |
| Next LP | `handoff_contexts[].consumed_by[]` | `[<next LP id>]` |
| Modified files list | `change_events[].summary` | Record only |
| README `source_root` | `meta.source_root` | Persist on scaffold AND direct-write (omit key if README value is placeholder/invalid) |
| README `scope` | `meta.scope` | Persist on scaffold AND direct-write (omit if README had none) |
| README `pst_root` | `meta.pst_root` | Persist on scaffold AND direct-write (omit if README had none) |
| README `coding_standards` (front-matter or `## Coding Standards` body section) | `meta.coding_standards` | Persist on scaffold AND direct-write (omit if both sources empty). Body section wins over front-matter when both exist. |

**Facts/Constraints format:** Each fact and constraint MUST be a structured object, not a plain string:

```yaml
# ✅ Correct
facts:
- id: "F-001"
  statement: "NetworkLiaison uses prediction buffer size of 64"
  source: "prompts/landing/LP-001-network-init.md"
  confidence: "high"

# ❌ Wrong (legacy format, do not use)
facts:
- "NetworkLiaison uses prediction buffer size of 64"
```

ID assignment for facts: `F-001`, `F-002`, ... (sequential within this HC).
ID assignment for constraints: `C-001`, `C-002`, ... (sequential within this HC).

### HC Management

**ID assignment:**
- Existing HC for this LP in status.yaml → update facts/constraints
- No existing HC → assign next sequential ID (HC-001, HC-002... based on max existing)
- New HC marked `status: available`

**`consumed_by` validation (closes dangling-consumer gap):**
- ELP MUST resolve every candidate `consumed_by` id against the in-memory `artifacts[]` snapshot loaded in Phase A Step 2.
- If the next-LP token resolves to an **already-registered** artifact id → include it in `consumed_by[]` and create a matching `consumed_status[]` entry with `status: pending`.
- If the next-LP token does NOT yet have a registered artifact (PSS only scaffolded the current LP, or the token is a free-form name) → DO NOT include the unresolved id in `consumed_by[]`. Instead add it under `pending_consumers: ["<token>"]` on the HC (PST §6G permits this optional field). PST AUDIT backfills `consumed_by[]` once the artifact is registered.
- If the current LP is the last token in the LP 序列 → both arrays empty.

This prevents `propagate.py` from chasing references to artifacts that do not exist yet.

**Version bump (only when ALL of these are true):**
- HC already exists
- facts or constraints content actually changed (compare `statement` fields)

### Direct-Write Meta Refresh

When using direct-write mode (no apply_changes.py), ELP MUST also update the `meta` block after modifying status.yaml:

```yaml
meta:
  last_run: "<current ISO timestamp>"
  last_updated: "<current ISO timestamp>"
  # Mirror README front-matter so PST render can regenerate the landing README
  # without losing ELP configuration. Omit a key if README has no value for it.
  source_root: "<from README front-matter>"
  scope:
    - "<from README front-matter>"
  pst_root: "<from README front-matter>"
  coding_standards: "<from README body section preferred, else front-matter; omit if both absent>"
  summary:
    artifacts_total: <recount>
    artifacts_ready: <recount>
    artifacts_blocked: <recount>
    artifacts_needs_update: <recount>
    blockers_open: <recount>
    gates_failed: <recount>
    handoffs_total: <recount>
    handoffs_stale: <recount>
    handoffs_invalidated: <recount>
    handoffs_pending_consumers: <recount>
```

This ensures PST's AGENTS.md and views reflect current state even before a full PST AUDIT runs.

### Idempotency

- Same LP re-executed: update existing artifact status, conditionally bump HC version, append new change_event
- Never create duplicate artifact or HC entries
- change_events always append (audit log)
- If result identical to current status.yaml state AND HC content unchanged → still append change_event (record execution fact), but don't modify artifact/HC

### change_event Format

```yaml
- id: CE-<next>
  time: "<ISO>"
  source: Execute-LandingPrompt
  event_type: lp_execution
  summary: "ELP: <one-line summary>"
  affected: [<artifact id>]
  transitions:
  - artifact: <artifact id>
    from: <previous_status or null>
    to: <ready|needs_update|blocked>
    reason: "ELP: <one-line summary>"
```

### § Sequential ID Generation (direct-write path)

When ELP writes status.yaml directly (no apply_changes.py), it MUST generate the next sequential id for `CE-`, `F-`, `C-`, and `HC-` prefixed entries using this algorithm:

```
next_id(existing_ids, prefix) =
  let used = { int(x[len(prefix):]) for x in existing_ids if x.startswith(prefix) and x[len(prefix):].isdigit() }
  let n = 1
  while n in used: n += 1
  return prefix + zero_pad(n, 3)         # e.g. "CE-007"
```

- `CE-NNN`: scan `change_events[*].id` across the entire file
- `HC-NNN`: scan `handoff_contexts[*].id` across the entire file
- `F-NNN` / `C-NNN`: scan `facts[*].id` / `constraints[*].id` **within the target HC only** (per-HC namespace, not global)

This matches `apply_changes.py`'s `next_id` helper, so direct-write and tool-path writes are interchangeable across runs and produce contiguous, non-colliding ids regardless of which path was used to author preceding entries.
