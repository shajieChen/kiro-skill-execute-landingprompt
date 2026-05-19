# Execute-LandingPrompt — Dependency Gate (companion)

Loaded on-demand by ELP SKILL.md Phase A Step 2 when the LP has prerequisites to verify. Discard from working memory once the gate has either passed or the user has chosen an override.

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

