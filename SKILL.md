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
lp_sequence_source: "user" | "auto"  # Optional, defaults to "user" if absent.
                                   # "auto"  → PST render may regenerate `## LP 序列` from topo sort.
                                   # "user"  → PST render MUST preserve the existing `## LP 序列` verbatim.
                                   # Set to "user" the moment a human edits `## LP 序列`.
                                   # ELP itself does not modify this field; it only reads `## LP 序列`.
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

**Important:** The `## LP 序列` section is authoritative for execution order. ELP always reads it verbatim. PST render preservation is governed by the `lp_sequence_source` front-matter flag (see PST §7 README generation contract): `"user"` → never overwritten; `"auto"` → may be regenerated. Missing flag is treated as `"user"` (fail-safe).

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

### Phase A — Execute (Steps 1–9.5)

1. Derive `lp_dir` from LP file path. Read `lp_dir/README.md`.
   - **Read cache**: compute sha256 of README.md; if `<pst_root>/status/.cache/readme_parse.json` exists and its stored hash matches, reuse the cached `{source_root, scope, pst_root, coding_standards, lp_sequence, lp_sequence_source}` tuple and skip re-parsing. Otherwise parse and overwrite the cache entry.
   - Parse YAML front-matter → extract `source_root`, `scope`, `pst_root`, `coding_standards`, `lp_sequence_source`
   - Validate `source_root`: must not be empty, placeholder, or nonexistent path
   - Parse body → extract LP sequence, Coding Standards (body section wins over front-matter per § Coding Standards Resolution)
   - If README.md missing → error and STOP
   - If source_root invalid → error and STOP
2. **Dependency Gate** — see § Dependency Gate. Run this **before** reading the current LP file content, **before** opening any prerequisite, and **before** touching any source file. If unsatisfied, STOP and ask the user; do not proceed until the user explicitly chooses an override.
   - **Batched read**: load `<pst_root>/status/status.yaml` **once** into an in-memory object and query `artifacts` / `preconditions` / `gates` / `handoff_contexts` / `blockers` against that single snapshot. Do not re-open the file per lookup.
3. Read the user-specified current LandingPrompt file.
4. Read prerequisite prompts only when needed for safe execution.
5. Execute only the current prompt (summarize goal/mode/allowed files/gates first).
6. Run relevant verification; compare against acceptance criteria.
7. Mark as `completed` / `partial` / `blocked`.
8. Read the next prompt (determined from LP sequence) for handoff context — do NOT execute it.
9. Produce the handoff section (Markdown).
9.5. **Result 持久化** — Write the complete Handoff Markdown to `<pst_root>/Result/<artifact-id>-<YYYYMMDD-HHmmss>.md`.
     - Create `<pst_root>/Result/` if it does not exist.
     - Content = everything from `## 当前 Prompt 执行结果` through `## 给下一个 Prompt 的交接` (inclusive).
     - Timestamp uses local time, format `YYYYMMDD-HHmmss` (no colons — Windows-safe).
     - Write failure → append `⚠️ Result 写入失败: <error>` to Handoff footer. Do NOT block Phase B or alter Phase A status.

### Phase B — PST 回流 (Steps 10–13)

10. Check if `<pst_root>/status/status.yaml` exists. If missing → scaffold (see § PST 回流).
11. Resolve LP artifact ID (see § ID Resolution).
12. Write/update: LP artifact entry + handoff_context + change_event.
13. Set LP artifact status (see § Status Mapping).

Phase B is best-effort. If YAML write fails, report the error in the handoff footer but do NOT alter Phase A results.

---

## § Dependency Gate (external)

Hard precondition check that runs as **Phase A Step 2**. Do not read prerequisite files, do not execute, do not edit source until this gate has either passed or the user has explicitly chosen an override.

Full algorithm (Step G.1 prereq sources, G.2 satisfaction table, G.3 user-override prompt, G.4 pass-through, Phase B interaction) lives in a sibling file. **Load on demand when entering Phase A Step 2:**

```
view C:\Users\chenshajie\.kiro\skills\Execute-LandingPrompt\dependency_gate.md
```

**Quick rules (no companion load required — apply in order; first match wins):**

1. **Greenfield trivial path:** `<pst_root>/status/status.yaml` is missing AND the LP file has no `Prereq` / `前置` / `Depends on` header AND the LP token is first in `## LP 序列` → gate auto-passes.

2. **All-satisfied fast path (closes the common-case load):** `status.yaml` exists AND the in-memory snapshot from Phase A Step 2 batched-read shows:
   - `artifacts[*] WHERE id == <current LP id>` exists AND its `depends_on[]` entries are all `status ∈ {ready, approved, archived}`, AND
   - `preconditions[*] WHERE target == <current LP id>` is either empty OR every row has `status == passing`, AND
   - `gates[*]` covering the current LP (per §6C generation rule) is either absent OR `status == passing` with all `checks[*].status == passing`, AND
   - every HC in `artifacts[current].consumes_handoffs[]` has `status ∈ {available, consumed}` AND either:
     - no `consumed_status[*] WHERE consumer == <current LP id>` row exists (first consumption, acceptable), OR
     - the row has `consumed_version == handoff.version` OR `consumed_version == null` (pending first consumption), AND
   - no `blockers[*] WHERE status == "open" AND <current LP id> ∈ blocks`.

   → gate auto-passes; record evidence inline as `依赖门结果: passed (fast path: all PCs/HCs/gates green, no open blockers)` and skip the companion load.

3. **Anything else** → load `dependency_gate.md` and run the full G.1–G.4 algorithm.

The fast path is intentionally strict: any ambiguity, missing field, or borderline status MUST fall through to the full algorithm. This keeps the per-LP-execution wire cost near-zero for the common "all green" case while preserving full rigor for the audit-required cases.

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
- Result 文件: <pst_root>/Result/<artifact-id>-<YYYYMMDD-HHmmss>.md

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

## § PST 回流 (external)

After Phase A completes (regardless of completed/partial/blocked), immediately execute Phase B (Steps 10-13). Full mechanics live in a sibling file. **Load on demand when entering Phase B:**

```
view C:\Users\chenshajie\.kiro\skills\Execute-LandingPrompt\phase_b.md
```

The companion covers: Write Method dispatch (apply_changes.py vs direct-write), Scaffold Rules (when status.yaml is missing), approved_transitions.json + handoff_register / handoff_version op schema, Data Mapping table, HC Management (including consumed_by validation), Direct-Write Meta Refresh, Idempotency, change_event format, and Sequential ID Generation algorithm.

Error Handling stays inline as `§ Phase B Error Handling` below — consult it on every Phase B failure regardless of whether the companion was loaded.

---

## § Phase B Error Handling

If Phase B fails at any step (YAML parse error, file write failure, lockfile contention, etc.):
1. Do NOT alter Phase A status determination.
2. **Enqueue the unwritten payload to `<pst_root>/status/.cache/pending_writebacks.json`** so that the next PST AUDIT can drain it (see PST §9). Algorithm:
   - Load existing `pending_writebacks.json` if present, else start `{"queue": []}`.
   - Append: `{"enqueued_at": <ISO>, "source": "Execute-LandingPrompt", "lp_file": "<absolute LP path>", "reason": "<short cause>", "payload": <the same JSON ELP intended to hand to apply_changes.py>}`.
   - Atomic write (write to `.tmp` then rename).
   - If the queue file itself cannot be written, surface a second failure in the handoff footer; do not retry indefinitely.
3. Append to handoff footer: `⚠️ PST 回流失败: <error>; 已入队 pending_writebacks (next AUDIT will drain).`
4. Continue normal handoff output.


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

