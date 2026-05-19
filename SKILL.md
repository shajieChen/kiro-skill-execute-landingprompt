---
name: Execute-LandingPrompt
description: "Execute exactly one LandingPrompt file from any project, then auto-sync execution results back to the project's status/status.yaml (PST回流). Project context (source root, coding standards, LP sequence) is read from the LandingPrompt directory's README.md. Use when the user says Skill + corresponding file, Skill + 对应文件, execute LandingPrompt, or asks to land one LandingPrompt step."
---

# Execute LandingPrompt

## Invocation

```text
Skill Execute-LandingPrompt + <path-to-any-landing-prompt-file>.md
```

Examples:
```text
Skill Execute-LandingPrompt + Q:\PortNotes\RB_Net_Monitor\LandingPrompt\CL_SubStep_Verify.md
Skill Execute-LandingPrompt + Q:\PortNotes\UE_Iris\LandingPrompt\Phase3-DescriptorRegistry.md
Skill Execute-LandingPrompt + prompts/landing/LP-001-some-task.md
```

If the user provides a folder, read that folder's `README.md` and ask which single prompt file to execute.

---

## § Path Derivation

Given user input `Skill Execute-LandingPrompt + <lp_file_path>`:

```
lp_dir            = dirname(lp_file_path)                           # LP 文件所在目录
readme_path       = lp_dir / README.md                              # README 位置
source_root       = README.front_matter.source_root                 # 必填，从 README 读取
scope             = README.front_matter.scope ?? [source_root, dirname(lp_dir)]
pst_root          = README.front_matter.pst_root ?? dirname(lp_dir)
coding_standards  = README.body["## Coding Standards"]
                    ?? README.front_matter.coding_standards         # body 优先；都没有则 None
```

**If `readme_path` does not exist:** Output error and STOP. Do not execute any LP without a README.

**If `source_root` is invalid:** If the value is a placeholder (e.g. `"<未配置>"`, empty string), or the path does not exist on disk, output error and STOP. A valid, accessible `source_root` is required for execution.

---

## § README Protocol

Every LandingPrompt directory MUST contain a `README.md` with this structure:

### YAML Front-matter (machine-readable config)

```yaml
---
source_root: "<absolute path>"     # REQUIRED: source code root directory
scope:                             # Optional: allowed file paths for ELP operations
  - "<path1>"                      # Default = [source_root, dirname(lp_dir)]
  - "<path2>"
pst_root: "<absolute path>"        # Optional: PST writeback target directory
                                   # Default = dirname(lp_dir)
                                   # ELP writes to <pst_root>/status/status.yaml
coding_standards: "<string>"       # Optional: project coding conventions
                                   # ELP also recognizes a `## Coding Standards`
                                   # body section (see § Coding Standards Resolution).
                                   # ELP mirrors this value into meta.coding_standards
                                   # so PST render can regenerate the README.
---
```

### Markdown Body (human-readable + ELP-parsed sections)

| Section heading | Purpose | Required |
|---|---|---|
| `## LP 序列` | Declares LP execution order; ELP uses this to determine "next prompt" | Yes |
| `## Coding Standards` | Project coding conventions; ELP follows these when modifying code | No |

Remaining body content is free-form for human readers.

**Important:** The `## LP 序列` section is authoritative for execution order. If PST auto-generates this README, ELP still uses the `## LP 序列` content as-is. If the auto-generated order is wrong, the user must correct it in the README (PST will preserve user-edited front-matter on subsequent renders).

### LP 序列解析算法

ELP 从 `## LP 序列` section 确定"下一个 prompt"：

1. **分割**：按 `->` 分割整个 section 内容（跨行 flatten 成一个 token 列表），trim 每个元素。
2. **匹配当前 LP**：将当前 LP 的 artifact ID（§ ID Resolution）和文件名 stem 与 token 列表比较（大小写不敏感），匹配规则任一即可：token == artifact ID / token == 文件名 stem / token 是其中之一的子串。
3. **取后继**：匹配 token 的下一个 token 即为"下一个 prompt"。
4. **映射回文件**：在 `lp_dir` 中查找文件名包含该 token 的 `.md`。多匹配时优先 `LP-\d{3}-` 前缀且 ID 匹配。
5. **末尾**：当前 LP 是序列最后一个 token → 无下一个 prompt（handoff 标注"序列已完成"）。

**示例（单行）**：`LP-001-init -> LP-002-core -> LP-003-verify` — 当前执行 LP-001 → 查找 `LP-002-core*.md`。

**多行格式**：所有行 `->` 链合并为扁平列表，解析逻辑不变。例：
```
CollectionLayer_Phase -> CL_SubStep_Verify -> CL_SubStep_Structures
TelemetryProcessingLayer_Phase -> TPL_SubStep_Framework -> TPL_SubStep_Snapshot
```

---

## § Coding Standards Resolution

ELP has no built-in coding standards. Reconciled with PST `meta.coding_standards` per PST §7.

| Read precedence (during execution) | Write-back to `meta.coding_standards` (Phase B) |
|---|---|
| 1. README body `## Coding Standards` | Canonical write target — body section wins if both sources exist |
| 2. README front-matter `coding_standards:` | Preserved verbatim by PST render, but meta is sourced from body when body exists |
| 3. PST `meta.coding_standards` | (fallback for read; no write-back needed) |
| 4. Workspace `.kiro/steering/` | — |
| 5. General best practices | — |

If neither README source has content, omit `meta.coding_standards` (do not write empty string).

---

## Execution Flow

### Phase A — Execute (Steps 1–9)

1. Derive `lp_dir` from LP file path. Read `lp_dir/README.md`.
   - Parse YAML front-matter → extract `source_root`, `scope`, `pst_root`, `coding_standards`
   - Validate `source_root`: must not be empty, placeholder, or nonexistent path
   - Parse body → extract LP sequence, Coding Standards (body section wins over front-matter per § Coding Standards Resolution)
   - If README.md missing → error and STOP
   - If source_root invalid → error and STOP
2. **Dependency Gate** — see § Dependency Gate. Run this **before** reading the current LP file content, **before** opening any prerequisite, and **before** touching any source file. If unsatisfied, STOP and ask the user; do not proceed until the user explicitly chooses an override.
3. Read the user-specified current LandingPrompt file.
4. Read prerequisite prompts only when needed for safe execution.
5. Execute only the current prompt (summarize goal/mode/allowed files/gates first).
6. Run relevant verification; compare against acceptance criteria.
7. Mark as `completed` / `partial` / `blocked`.
8. Read the next prompt (determined from LP sequence) for handoff context — do NOT execute it.
9. Produce the handoff section (Markdown).

### Phase B — PST 回流 (Steps 10–13)

10. Check if `<pst_root>/status/status.yaml` exists. If missing → scaffold (see § PST 回流).
11. Resolve LP artifact ID (see § ID Resolution).
12. Write/update: LP artifact entry + handoff_context + change_event.
13. Set LP artifact status (see § Status Mapping).

Phase B is best-effort. If YAML write fails, report the error in the handoff footer but do NOT alter Phase A results.

---

## § Dependency Gate

Hard precondition check that runs as Phase A Step 2. Do **not** read prerequisite files, do **not** execute, do **not** edit source until this gate has either passed or the user has explicitly chosen an override.

### Step G.1 — Identify prerequisites of the current LP

Collect prerequisites from these sources, in order of authority:

1. **PST `status.yaml`** at `<pst_root>/status/status.yaml` (if it exists). Use these **exact** lookups (PCs and gates are top-level, NOT nested under artifacts):
   - `artifacts[*] WHERE id == <current LP id>` → `.depends_on`, `.consumes_handoffs`, `.produces_handoffs`
   - `preconditions[*] WHERE target == <current LP id>` → each `.status` and its `.requires[]` clauses
   - `gates[*]` whose any `checks[*]` references the current LP, or whose name is associated with the current LP via PST §6C generation rule (one Gate per LP that has PCs) — read `.status` and each `checks[*].status`
   - `handoff_contexts[*]` where `id ∈ artifacts[current].consumes_handoffs` → `.status`, `.version`, and `.consumed_status[*] WHERE consumer == <current LP id>`
   - `blockers[*] WHERE <current LP id> ∈ blocks` AND `status == "open"`
   - Resolve current LP id via § ID Resolution
2. **Explicit fields inside the current LP file**: lines/sections labeled `Prereq`, `前置`, `Depends on`, `依赖`. Read only the header region of the LP file for this — do not yet read the body.
3. **README `## LP 序列`**: every token strictly earlier than the current LP token in the parsed flat sequence is an **implicit prerequisite**, with one exception: **phase main files** (filenames matching `*_Phase.md` or LP tokens that contain `_Phase`) do not inherit this implicit chain — they are treated as index/constraint files, not as sequential steps.

Deduplicate prerequisites across sources (PST id wins over file path wins over README token).

### Step G.2 — Resolve each prerequisite's status

For every prerequisite `P`:

| Source | Satisfied iff | Unsatisfied iff |
|--------|--------------|----------------|
| PST artifact | `status ∈ { ready, approved, archived }` | `status ∈ { draft, reviewed, needs_update, blocked, invalidated, deprecated }` or artifact missing |
| PST precondition (top-level, `target == current LP`) | `status == passing` | `status ∈ { failed, pending }` or status missing |
| PST gate (covers current LP) | `status == passing` AND every `checks[*].status == passing` | any check `failed` or gate `status ∈ { failed, pending }` |
| Handoff context | `status ∈ { available, consumed }` AND consumer's `consumed_status[*].consumed_version == handoff.version` | any other `status` (e.g. `draft`, `partially_consumed`, `stale`, `invalidated`, `deprecated`, `archived`) OR `consumed_version` mismatch OR no `consumed_status` entry for current LP |
| PST blocker | no open blocker has the current LP in `blocks[]` | any blocker with `status == open` lists the current LP in `blocks[]` |
| File evidence only | Latest "执行状态" marker in the handoff section equals `completed` | Marker equals `partial` / `blocked`, or marker absent |
| README implicit | Corresponding PST artifact or file evidence shows satisfied | Otherwise |

A failed `gate` or open `blocker` referencing the current LP is always **unsatisfied**, regardless of artifact status.

> **Note on PST schema:** `preconditions` and `gates` are **top-level arrays** in `status.yaml` (per PST §6G schema), keyed by `target` / `checks[*]` references. They are NOT nested fields of `artifacts[]`. Looking them up under `artifacts[id].preconditions` will always miss.

### Step G.3 — If any prerequisite is unsatisfied → STOP and ask the user

Do not read the current LP body in full, do not read prerequisite files in full, do not run verification commands, do not touch source. Emit exactly one question to the user using this template, then wait for the answer:

```text
当前目标 Prompt: <current LP file>
PST 上下文: <pst_root>/status/status.yaml (exists | missing)
当前 LP artifact id: <resolved id>

检测到前置依赖未满足：
  - <prereq id or token>: <status / reason / source>
  - <HC-xxx>: <stale | invalidated | not consumed by current LP, producer=<id>, version=<n>>
  - <gate id>: failed — <reason>
  - <blocker id>: open — <title>

请选择：
  (1) 强制执行当前 Prompt（接受风险：可能猜不存在的锚点、可能产生悬空引用、PST 回流仍会写入但 from 状态可能不准）
  (2) 改为先执行最近一个未完成前置：<suggested prereq file path>
  (3) 取消，先人工修复 <root cause>
  (4) 仅做 verify-only 子集（仅当当前 Prompt 显式声明 verify 模式时可用；ELP 强制 mode=verify-only，不会修改源码）
```

User responses map as:

| Choice | Skill behavior |
|--------|----------------|
| (1) Force execute | Record `dependency_override: forced` in the handoff result; continue with Step 3. **Phase B regardless of Phase A outcome maps the artifact status to `needs_update`** (never `ready`) because PCs are not verified — PST's invariant "ready requires PCs passed" must hold. The `change_events[].summary` MUST include `(forced override, PCs unverified)`. Phase A's true result (completed/partial/blocked) is still recorded in the handoff result block for audit. |
| (2) Switch target | Abort current invocation. Instruct user to re-invoke ELP with the suggested prereq file. Do not silently switch — ELP is single-target. Emit no Phase B writeback. |
| (3) Cancel | Abort. Emit only the dependency report (no Phase A execution, no Phase B writeback, no handoff for the current LP). |
| (4) Verify-only | Continue with Step 3 but force `mode = verify-only` regardless of what the LP declares; never edit source. Phase B writeback still runs and maps result to `needs_update` (verify-only cannot produce `ready`). |

### Step G.4 — If all prerequisites satisfied → proceed

Record the resolved prerequisites and their evidence (PST field path or file:line) in the handoff result under `## 前置依赖检查`. Set `依赖门结果: passed`.

### Interaction with Phase B

`change_events[].summary` always appends the gate outcome in parens: `passed` / `forced override` / `verify-only override` / `cancelled (no Phase B write)`. Choice (3) skips Phase B; (1)/(4) run Phase B normally with the override captured in `change_events`.

---

## Constraints

- **Single Prompt Rule**: Do not execute sibling prompts, do not widen scope, do not start the next prompt. If blocked, hand off the blocker.
- **Verify-only prompts**: Do not modify source files.
- **Missing anchors**: Stop and report, do not guess nonexistent functions/paths.
- **Allowed files only**: Keep edits inside the current prompt's file list AND within `scope`.
- **Preserve `禁止修改` / `不应改动` rules**.
- **`confirmed` = reusable facts; `unresolved` = gates to verify first**.
- **If an unresolved gate fails**: Stop, write handoff note, do not force code.

---

## Handoff Template

```markdown
## 当前 Prompt 执行结果

- 当前文件:
- 执行状态: completed / partial / blocked
- 依赖门结果: passed / forced / verify-only-override / cancelled
- 已完成:
- 未完成:
- 修改文件:
- 验证结果:

## 前置依赖检查

- 检查来源: status.yaml / LP 文件头 / README LP 序列
- 当前 LP artifact id:
- 前置项与状态:
  - <prereq id or token>: satisfied | unsatisfied (reason) | overridden
  - <HC-xxx>: available v<n> | stale | invalidated
- 用户选择: (1) 强制 / (2) 改目标 / (3) 取消 / (4) verify-only / N/A（全部满足）

## confirmed

- ...

## unresolved

- ...

## 给下一个 Prompt 的交接

- 下一个文件:
- 已预读但未执行:
- 下一个 Prompt 的入口条件:
- 当前阶段提供给它的可复用事实:
- 必须避免重复做的事:
- 必须先验证的风险:
```

---

## § ID Resolution

### LP Artifact ID

Algorithm per PST §6A, with one ELP-specific precedence override:

0. **PST-registered ID** (ELP-only, highest): if `<pst_root>/status/status.yaml` has an artifact whose `path` matches the LP file → reuse that `id` (protects against re-keying on re-execution).
1.–4. **Then fall back to PST §6A order**: filename `LP-\d{3}-*` → YAML front-matter `id:` → H1 `# LP-NNN:` → slug `LP.<stem>`.

If steps 1–4 disagree (e.g., filename `LP-001` vs H1 `LP-007`), pick the earliest match and emit a warning under `## 前置依赖检查`. Do not silently prefer one source.

### LP Path in status.yaml

| Source | `artifacts[].path` value |
|--------|--------------------------|
| Workspace-local (`prompts/landing/`) | `prompts/landing/<filename>.md` (relative to pst_root) |
| External (any other absolute path) | `external:<project_dir_name>/<filename>.md` |

The `external:` prefix tells PST dirty_check to skip this artifact during file-based scanning.

---

## § Status Mapping

Raw Phase A → PST status mapping. **Whitelisted transitions** (per PST §1 ELP privilege): `ready→ready`, `ready→needs_update`, `ready→blocked`, `needs_update→ready`, `blocked→ready`. Any other target/path is forbidden.

| ELP Result | Dependency Gate | Mapped PST status | Notes |
|------------|-----------------|-------------------|-------|
| completed | passed | `ready` | Outputs available; HC version bumps if facts/constraints content changed |
| completed | forced override | `needs_update` | PST invariant: ready requires PCs passed |
| completed | verify-only override | `needs_update` | Verify-only cannot mark ready |
| partial | any | `needs_update` | Re-execution needed |
| blocked | any | `blocked` | External resolution needed |

**Fallback when raw mapping violates whitelist** (`apply_changes.py` does NOT validate this — ELP enforces):

Retain current status unchanged, emit NO `transitions[]` row, but still append a `change_event` (audit log) with `summary` noting the forbidden combination. Common cases:

| Mapped want | Current status | Action |
|---|---|---|
| `needs_update` | `needs_update` | No transition; summary: "status unchanged (whitelist forbids needs_update→needs_update)" |
| `needs_update` | `blocked` | No transition; summary: "ELP attempted needs_update but whitelist forbids blocked→needs_update; user action required to unblock" |
| `blocked` | `needs_update` | No transition; summary notes blocker reason; handoff footer prompts "to formally mark blocked, run PST manual flow or let AUDIT propagate" |
| Any whitelisted combination | — | Write transition normally |

---

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

Or for updating an existing HC (version bump):

```json
{
  "op": "handoff_version",
  "handoff_id": "HC-<existing>",
  "version": "<new version number>",
  "to": "available",
  "reason": "ELP re-execution: facts/constraints changed",
  "source": "Execute-LandingPrompt"
}
```

> Note: `apply_changes.py` accepts either `handoff_id` (preferred) or `artifact` for backward compatibility. New ELP writes MUST use `handoff_id` only.

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

### Error Handling

If Phase B fails at any step (YAML parse error, file write failure, etc.):
1. Do NOT alter Phase A status determination
2. Append to handoff footer: `⚠️ PST 回流失败: <error description>`
3. Continue normal handoff output

---

## Safety Checks (before final response)

Only items not already enforced by inline `MUST` / `MUST NOT` clauses in the body. Body rules win on conflict.

- README.md read; front-matter parsed; `source_root` exists & is not a placeholder.
- Dependency Gate ran **as Phase A Step 2**, before any prerequisite or source file was opened.
- If any prerequisite unsatisfied, user explicitly chose (1)–(4); no silent continuation.
- Current prompt was the only one executed; next prompt was read but NOT executed.
- All edits within declared `scope`; no `禁止修改` regions touched; no anchors guessed.
- Phase B write method matches pst_root state (`apply_changes.py` if exists, else direct-write).
- Phase B: every transition object includes `"source": "Execute-LandingPrompt"` (the single critical flag — see § approved_transitions.json Format).
- Phase B 回流完成或失败已报告.
