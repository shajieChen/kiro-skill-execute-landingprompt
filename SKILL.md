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
lp_dir       = dirname(lp_file_path)                           # LP 文件所在目录
readme_path  = lp_dir / README.md                              # README 位置
source_root  = README.front_matter.source_root                 # 必填，从 README 读取
scope        = README.front_matter.scope ?? [source_root, dirname(lp_dir)]
pst_root     = README.front_matter.pst_root ?? dirname(lp_dir)
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

ELP does NOT contain any built-in coding standards. Resolution priority:

1. README.md `## Coding Standards` section → if present, follow those rules
2. If README has no such section → follow workspace `.kiro/steering/` rules (if any)
3. If neither exists → follow general best practices

---

## Execution Flow

### Phase A — Execute (Steps 1–8)

1. Derive `lp_dir` from LP file path. Read `lp_dir/README.md`.
   - Parse YAML front-matter → extract `source_root`, `scope`, `pst_root`
   - Validate `source_root`: must not be empty, placeholder, or nonexistent path
   - Parse body → extract LP sequence, Coding Standards
   - If README.md missing → error and STOP
   - If source_root invalid → error and STOP
2. Read the user-specified current LandingPrompt file.
3. Read prerequisite prompts only when needed for safe execution.
4. Execute only the current prompt (summarize goal/mode/allowed files/gates first).
5. Run relevant verification; compare against acceptance criteria.
6. Mark as `completed` / `partial` / `blocked`.
7. Read the next prompt (determined from LP sequence) for handoff context — do NOT execute it.
8. Produce the handoff section (Markdown).

### Phase B — PST 回流 (Steps 9–12)

9. Check if `<pst_root>/status/status.yaml` exists. If missing → scaffold (see § PST 回流).
10. Resolve LP artifact ID (see § ID Resolution).
11. Write/update: LP artifact entry + handoff_context + change_event.
12. Set LP artifact status (see § Status Mapping).

Phase B is best-effort. If YAML write fails, report the error in the handoff footer but do NOT alter Phase A results.

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
- 已完成:
- 未完成:
- 修改文件:
- 验证结果:

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

Resolution order (first match wins):

1. **PST-registered ID**: If `status/status.yaml` already contains an artifact whose `path` matches the LP file → use that artifact's `id`.
2. **Filename pattern**: If filename matches `LP-\d{3}-*.md` → extract `LP-001` etc.
3. **Fallback slug**: `LP.<stem>` (filename without `.md`).

### LP Path in status.yaml

| Source | `artifacts[].path` value |
|--------|--------------------------|
| Workspace-local (`prompts/landing/`) | `prompts/landing/<filename>.md` (relative to pst_root) |
| External (any other absolute path) | `external:<project_dir_name>/<filename>.md` |

The `external:` prefix tells PST dirty_check to skip this artifact during file-based scanning.

---

## § Status Mapping

| ELP Result | PST status | Semantics |
|----------|-----------|------|
| completed | `ready` | LP executed successfully, outputs available for downstream |
| partial | `needs_update` | LP partially completed, needs re-execution |
| blocked | `blocked` | LP blocked, needs external resolution |

**Valid re-execution transitions:**
- `ready → ready` (re-executed, still completed — HC version bumps if content changed)
- `ready → needs_update` / `ready → blocked`
- `needs_update → ready` / `needs_update → needs_update`
- `blocked → ready` / `blocked → needs_update`

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
  "artifact": "HC-<existing>",
  "handoff_id": "HC-<existing>",
  "version": "<new version number>",
  "to": "available",
  "reason": "ELP re-execution: facts/constraints changed",
  "source": "Execute-LandingPrompt"
}
```

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
- Phase B: `meta.source_root` / `meta.scope` / `meta.pst_root` mirror README front-matter so a later PST AUDIT can regenerate the landing README without losing ELP configuration.
- Phase B 回流完成或失败已报告.
