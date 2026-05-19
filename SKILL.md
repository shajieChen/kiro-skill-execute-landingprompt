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

ELP 按以下规则从 `## LP 序列` section 确定"下一个 prompt"：

1. **分割**：按 `->` 分割整个 section 内容（跨行），trim 每个元素得到 token 列表。
2. **匹配当前 LP**：将当前 LP 的 artifact ID（按 § ID Resolution 解析）和文件名 stem 与 token 列表逐一比较（大小写不敏感）。匹配规则：
   - token == artifact ID（如 `LP-001`）
   - token == 文件名 stem（如 `CL_SubStep_Verify`）
   - token 是 artifact ID 的子串或文件名 stem 的子串
3. **取后继**：匹配到的 token 的下一个 token 即为"下一个 prompt"。
4. **映射回文件**：在 `lp_dir` 中查找文件名包含该 token 的 `.md` 文件。如果有多个匹配，优先选择以 `LP-\d{3}-` 开头且 ID 匹配的文件。
5. **末尾处理**：如果当前 LP 是序列中最后一个 token，则无下一个 prompt（handoff 中标注"序列已完成"）。

**示例**：
```
## LP 序列

LP-001-init -> LP-002-core -> LP-003-verify
```
当前执行 `LP-001-init.md`，匹配 token `LP-001-init`，下一个 token 为 `LP-002-core`，在 lp_dir 中查找 `LP-002-core*.md`。

多行格式同样支持：
```
## LP 序列

CollectionLayer_Phase -> CL_SubStep_Verify -> CL_SubStep_Structures
TelemetryProcessingLayer_Phase -> TPL_SubStep_Framework -> TPL_SubStep_Snapshot
```
所有行的 token 合并为一个扁平列表（按出现顺序），解析逻辑不变。

---

## § Coding Standards Resolution

ELP does NOT contain any built-in coding standards. Two sources are recognized and reconciled with PST `meta.coding_standards` per PST §7:

**Read precedence (what rules to follow during execution):**

1. README.md `## Coding Standards` body section → if present, follow those rules
2. README.md front-matter `coding_standards:` field → if present (and no body section), follow those rules
3. PST `meta.coding_standards` (from `<pst_root>/status/status.yaml`) → if neither README source is present, use the meta value
4. Workspace `.kiro/steering/` rules (if any)
5. General best practices

**Write-back to meta (Phase B):**

- If README front-matter or body provides any coding_standards content, ELP MUST persist the resolved string into `meta.coding_standards` on both scaffold and direct-write meta refresh.
- If multiple sources exist, the body section is canonical and is the value written to meta. Front-matter is preserved verbatim by PST render, but meta is the round-trip source of truth.
- If neither README source has content, omit the `meta.coding_standards` key (do not write empty string).

This closes the loop with PST §7's README generation: PST writes `meta.coding_standards` into the regenerated README, ELP reads README and mirrors back to meta — no field ever gets lost in a round trip.

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
| PST precondition (top-level, `target == current LP`) | `status == passed` | `status ∈ { failed, pending }` or status missing |
| PST gate (covers current LP) | `status == passed` AND every `checks[*].status == passed` | any check `failed` or gate `status ∈ { failed, pending }` |
| Handoff context | `status == available` AND consumer's `consumed_status[*].consumed_version == handoff.version` | `status ∈ { stale, invalidated }` OR consumed_version mismatch OR no `consumed_status` entry for current LP |
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

### Interaction with Phase B (PST 回流)

- Phase B always writes the `change_events[].summary` with the dependency-gate outcome appended in parentheses: `passed` / `forced override` / `verify-only override` / `cancelled (no Phase B write)`.
- When user chose (3) Cancel, Phase B does NOT run.
- When user chose (1) or (4), Phase B runs normally; the override is captured in `change_events` for audit.

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

Resolution order (first match wins). Algorithm aligned with PST §6A, with PST-registered ID checked first so existing artifacts are never re-keyed:

1. **PST-registered ID**: If `<pst_root>/status/status.yaml` already contains an artifact whose `path` matches the LP file → use that artifact's `id` (highest precedence; protects against ID drift on re-execution).
2. **Filename pattern**: If filename matches `LP-\d{3}-*.md` → extract `LP-001`, `LP-002`, etc.
3. **YAML front-matter `id:` field**: If the LP file's YAML front-matter contains an `id:` field with a string value → use that value verbatim.
4. **H1 prefix**: If the LP file's first H1 line matches `# (LP-\d{3})[:\s].*` → extract the captured `LP-NNN` token.
5. **Fallback slug**: `LP.<stem>` (filename without `.md`, no further normalization).

If resolution steps 2–5 disagree (e.g., filename says `LP-001` but H1 says `LP-007`), pick the **earliest** match in the order above and emit a warning in the handoff result under `## 前置依赖检查` so the user can reconcile. Do not silently prefer one source over another beyond the documented order.

### LP Path in status.yaml

| Source | `artifacts[].path` value |
|--------|--------------------------|
| Workspace-local (`prompts/landing/`) | `prompts/landing/<filename>.md` (relative to pst_root) |
| External (any other absolute path) | `external:<project_dir_name>/<filename>.md` |

The `external:` prefix tells PST dirty_check to skip this artifact during file-based scanning.

---

## § Status Mapping

| ELP Result | Dependency Gate outcome | PST status | Semantics |
|------------|-------------------------|------------|----------|
| completed | passed | `ready` | LP executed successfully with verified preconditions; outputs available for downstream |
| completed | forced override | `needs_update` | LP body finished but preconditions unverified — must NOT promote to `ready` (PST invariant) |
| completed | verify-only override | `needs_update` | Verify-only by definition cannot mark `ready` |
| partial | any | `needs_update` | LP partially completed, needs re-execution |
| blocked | any | `blocked` | LP blocked, needs external resolution |

**Valid re-execution transitions (must match PST §1 ELP accommodations exactly):**

- `ready → ready` (re-executed, still completed — HC version bumps if content changed)
- `ready → needs_update`
- `ready → blocked`
- `needs_update → ready`
- `blocked → ready`

**Disallowed transitions — ELP MUST NOT write these to `transitions[]`:**
- `needs_update → needs_update`
- `blocked → needs_update`
- `needs_update → blocked`
- Any transition into `draft`, `reviewed`, `approved`, `invalidated`, `deprecated`, `archived`

**"Wanted but forbidden" fallback matrix.** When Phase A's raw mapping (per the Status Mapping table) would produce a forbidden transition given the current PST status, ELP MUST fall back as follows. In every fallback case, the artifact `status` is **retained unchanged** and the `transitions[]` array is **left empty** for that artifact; the execution is still recorded as a `change_event` (audit log) with the situation noted in `summary`.

| Phase A raw mapping | Current PST status | Direct write legal? | ELP fallback |
|---|---|---|---|
| `needs_update` (partial / forced / verify-only) | `needs_update` | ❌ | No transition row; `change_event.summary` notes "status unchanged (PST whitelist: needs_update→needs_update forbidden)" |
| `needs_update` (partial / forced / verify-only) | `blocked` | ❌ | No transition row; status retained as `blocked`; summary notes "ELP attempted needs_update mapping but PST whitelist forbids blocked→needs_update; user action required to unblock" |
| `blocked` (Phase A blocked) | `needs_update` | ❌ (`needs_update→blocked` not whitelisted) | No transition row; status retained as `needs_update`; summary notes blocker reason; handoff footer prompts "to formally mark blocked, run PST manual flow needs_update→reviewed→…→blocked, or let PST AUDIT set blocked via propagate" |
| Any other combination | — | ✅ (in whitelist) | Write transition normally |

**Rationale:** `apply_changes.py` does NOT validate state-machine legality (verified by audit). ELP is contractually responsible for emitting only whitelisted transitions. The fallback rule above preserves audit completeness without violating the PST §1 invariant.

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

Minimal status.yaml:

```yaml
meta:
  project_name: "<pst_root folder name>"
  schema_version: 1
  created: "<ISO timestamp>"
  last_updated: "<ISO timestamp>"
  last_run: "<ISO timestamp>"
  # ELP MUST persist the resolved README front-matter values into meta so a
  # subsequent PST AUDIT can regenerate the landing README without losing
  # ELP-required configuration. If a value is absent in README, omit the key
  # (do NOT write the "<未配置>" placeholder into meta).
  source_root: "<resolved source_root>"
  scope:                                   # omit if README had no scope
    - "<resolved scope entry>"
  pst_root: "<resolved pst_root>"
  coding_standards: "<resolved coding_standards>"   # omit if README had no body section AND no front-matter field
  summary:
    artifacts_total: 1
    artifacts_ready: 0
    artifacts_blocked: 0
    artifacts_needs_update: 0
    blockers_open: 0
    gates_failed: 0
    handoffs_total: 1
    handoffs_stale: 0
    handoffs_invalidated: 0
    handoffs_pending_consumers: 1
  hotspots: []
  pointers:
    entry_point: "AGENTS.md"
    views_dir: "views/"
    status_report: "views/status_report.md"
    handoff_view: "views/handoff_view.md"
    prompt_chain: "views/prompt_chain_view.md"

project:
  name: "<pst_root folder name>"
  phase: "execution"
  version: "1.0"

artifacts:
- id: <resolved per § ID Resolution>
  type: landing_prompt
  path: <resolved per § ID Resolution path rules>
  status: <ready|needs_update|blocked>
  depends_on: []
  produces_handoffs: [HC-001]
  consumes_handoffs: []
  last_checked: "<ISO timestamp>"

research_findings: []
evidence: []
assumptions: []
decisions: []
dependencies: []

rules:
  research_is_fact_only: true
  status_is_single_source_of_truth: true
  landing_requires_test_ready: true
  plan_invalidates_landing: true
  landing_invalidates_test: true

handoff_contexts:
- id: HC-001
  title: "<LP title or filename>"
  producer: <artifact id>
  producer_type: landing_prompt
  produced_from: []
  version: 1
  status: available
  results: []
  invalidated_by: []
  facts:
  - id: "F-001"
    statement: "<confirmed item 1>"
    source: "<lp_file_path>"
    confidence: "high"
  constraints:
  - id: "C-001"
    statement: "<unresolved item 1>"
    source: "<lp_file_path>"
  consumed_by: [<next LP id>]
  consumed_status:
  - consumer: <next LP id>
    status: "pending"
    consumed_version: null
    consumed_at: null
  last_verified: "<ISO timestamp>"

blockers: []
gates: []
preconditions: []

snapshots:
  git_baseline: null
  file_hashes: {}

change_events:
- id: CE-001
  time: "<ISO>"
  source: Execute-LandingPrompt
  event_type: scaffold_and_execute
  summary: "ELP scaffold + first execution: <one-line summary>"
  affected: [<artifact id>]
  transitions:
  - artifact: <artifact id>
    from: null
    to: <ready|needs_update|blocked>
    reason: "ELP execution: <one-line summary>"
```

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

### Error Handling

If Phase B fails at any step (YAML parse error, file write failure, etc.):
1. Do NOT alter Phase A status determination
2. Append to handoff footer: `⚠️ PST 回流失败: <error description>`
3. Continue normal handoff output

---

## Safety Checks (before final response)

- README.md was read and front-matter parsed successfully.
- source_root path exists, is accessible, and is not a placeholder value.
- **Dependency Gate (§ Dependency Gate) ran as Phase A Step 2, before any prerequisite or source file was opened.**
- **If any prerequisite was unsatisfied, the user was asked and explicitly chose one of (1)–(4); no silent continuation.**
- **Handoff result includes `依赖门结果` field and a `## 前置依赖检查` block listing every resolved prerequisite.**
- **If `dependency_override: forced` was recorded, `change_events[].summary` includes `(forced override)` and the report explicitly lists which anchors were assumed rather than verified.**
- **If user chose (3) Cancel, Phase B did NOT run.**
- Current prompt was read and only it was executed.
- Next prompt was read but not executed; handoff section exists.
- All code modifications are within declared scope.
- Code follows Coding Standards declared in README (or steering fallback).
- No forbidden scope modified; no unresolved reported as confirmed.
- Phase B: artifact ID resolved correctly (PST-registered > filename pattern > fallback).
- Phase B: status mapping uses `ready`/`needs_update`/`blocked` (never `archived`).
- Phase B: write method matches pst_root state (apply_changes.py if exists, else direct).
- Phase B: HC facts/constraints use structured format `{id, statement, source, confidence}`.
- Phase B: HC version only bumped if facts/constraints actually changed.
- Phase B: When using apply_changes.py path, every transition includes `"source": "Execute-LandingPrompt"`.
- Phase B: When using direct-write, meta block is refreshed after write.
- Phase B: `meta.source_root` / `meta.scope` / `meta.pst_root` / `meta.coding_standards` mirror README (front-matter or body section as applicable) so a later PST AUDIT can regenerate the landing README without losing ELP configuration.
- Phase B: Forced-override executions map to `needs_update` regardless of Phase A success (PST's "ready requires PCs passed" invariant must not be violated).
- Phase B 回流完成或失败已报告.
