# Execute-LandingPrompt

Kiro Agent Skill — 执行单个 LandingPrompt 文件并自动回流结果到 `status.yaml`（PST 回流）。

## 三件套协作关系

### 全局定位

```
+===========================================================================+
|                    ELP 在三件套中的角色: [执行者]                            |
+===========================================================================+
|                                                                           |
|  +---------------------+                                                  |
|  | project-state-spec  |  生成 LP 文件 (PSS 的产出)                        |
|  |       (PSS)         |  定义 LP 序列、AC、架构                           |
|  |     [规划者]         |                                                  |
|  +----------+----------+                                                  |
|             |                                                             |
|             | LP 文件写入磁盘, 注册到 status.yaml                           |
|             v                                                             |
|  +-------------------------------------------------------+                |
|  |            project-state-tracker (PST)                 |                |
|  |                   [管家]                                |                |
|  |                                                       |                |
|  |  status.yaml 存储:                                     |                |
|  |  - artifacts[]: LP 的状态 + depends_on                 |                |
|  |  - preconditions[]: LP 的前置条件                       |                |
|  |  - gates[]: LP 的质量门                                 |                |
|  |  - handoff_contexts[]: LP 间的交接上下文                 |                |
|  |  - blockers[]: 阻塞项                                  |                |
|  +----------------------------+--------------------------+                |
|                               |                                           |
|           读取 (Phase A Step 2 依赖门)                                     |
|                               |                                           |
|                               v                                           |
|  +------------------------------+                                         |
|  |  Execute-LandingPrompt (ELP) |  <-- 你在这里                            |
|  |         [执行者]              |                                         |
|  |                              |  输入: 单个 LP 文件 + README 上下文       |
|  |  Phase A: 依赖门 -> 执行 LP   |  输出: 代码修改 + Result 归档            |
|  |  Phase B: 回流到 status.yaml  |  回流: ready/needs_update/blocked       |
|  |  Result/: 执行历史归档        |                                         |
|  +------------------------------+                                         |
|             |                                                             |
|             | Phase B 写入                                                 |
|             v                                                             |
|  +-------------------------------------------------------+                |
|  |  PST (status.yaml 更新)                                |                |
|  |  - artifact status 变更                                |                |
|  |  - HC 注册/版本更新                                     |                |
|  |  - change_event 追加                                   |                |
|  +-------------------------------------------------------+                |
|                                                                           |
+===========================================================================+
```

### ELP 执行流程详解

```
+-----------------------------------------------------------------------+
|                    ELP 单次执行的完整流程                                |
+-----------------------------------------------------------------------+
|                                                                       |
|  [输入] 用户: Skill Execute-LandingPrompt + <lp_file>.md              |
|     |                                                                 |
|     v                                                                 |
|  +-- Phase A Step 1: 读取 README.md --------------------------------+|
|  |   - 解析 front-matter: source_root, scope, pst_root               ||
|  |   - 解析 body: ## LP 序列, ## Coding Standards                     ||
|  |   - 验证 source_root 有效性                                        ||
|  +-------------------------------------------------------------------+|
|     |                                                                 |
|     v                                                                 |
|  +-- Phase A Step 2: 依赖门检查 ------------------------------------+|
|  |   读取 status.yaml (batched, 单次加载):                            ||
|  |   - artifacts[current].depends_on --> 前置工件状态                  ||
|  |   - preconditions[target=current] --> PC 是否 passing              ||
|  |   - gates[covers current] --> 门是否 passing                       ||
|  |   - handoff_contexts[consumed] --> HC 版本是否匹配                  ||
|  |   - blockers[blocks current] --> 是否有 open blocker               ||
|  |                                                                    ||
|  |   结果:                                                            ||
|  |   - 全部满足 --> passed (继续)                                     ||
|  |   - 任一不满足 --> STOP, 询问用户 (1)强制/(2)改目标/(3)取消/(4)verify||
|  +-------------------------------------------------------------------+|
|     |                                                                 |
|     v                                                                 |
|  +-- Phase A Steps 3-9.5: 执行 LP ----------------------------------+|
|  |   - 读取 LP 文件内容                                               ||
|  |   - 执行实施任务 (在 scope 内修改代码)                              ||
|  |   - 运行验证, 对照 acceptance criteria                             ||
|  |   - 判定: completed / partial / blocked                            ||
|  |   - 读取下一个 LP (仅预读, 不执行)                                  ||
|  |   - 生成 Handoff Markdown                                          ||
|  |   - 写入 Result/<artifact-id>-<timestamp>.md                       ||
|  +-------------------------------------------------------------------+|
|     |                                                                 |
|     v                                                                 |
|  +-- Phase B Steps 10-13: PST 回流 ---------------------------------+|
|  |   写入方式:                                                        ||
|  |   - tools/apply_changes.py 存在 --> approved_transitions.json      ||
|  |   - tools/ 不存在 --> direct-write status.yaml                     ||
|  |   - status.yaml 不存在 --> scaffold 最小结构 + direct-write         ||
|  |                                                                    ||
|  |   写入内容:                                                        ||
|  |   - artifact status: ready / needs_update / blocked                ||
|  |   - handoff_context: HC 注册或版本更新                              ||
|  |   - change_event: 执行记录 (含 source: Execute-LandingPrompt)      ||
|  |   - meta refresh: source_root, coding_standards 等                 ||
|  |                                                                    ||
|  |   失败处理:                                                        ||
|  |   - 写入失败 --> 入队 pending_writebacks.json                       ||
|  |   - 下次 PST AUDIT 会 drain 该队列                                 ||
|  +-------------------------------------------------------------------+|
|                                                                       |
+-----------------------------------------------------------------------+
```

### ELP 与 PST 的交互协议

```
ELP 从 PST 读取 (Phase A):
  +-- status.yaml:
  |     artifacts[*].status, .depends_on, .consumes_handoffs
  |     preconditions[*].status, .requires[]
  |     gates[*].status, .checks[*].status
  |     handoff_contexts[*].status, .version, .consumed_status[]
  |     blockers[*].status, .blocks[]

ELP 向 PST 写入 (Phase B):
  +-- approved_transitions.json (当 apply_changes.py 存在):
  |     transitions[]: {artifact, from, to, reason, source:"Execute-LandingPrompt"}
  |     handoff ops: {op:"handoff_register"/"handoff_version", ...}
  |
  +-- status.yaml direct-write (当 tools/ 不存在):
  |     artifacts[].status 直接修改
  |     handoff_contexts[] 直接追加/更新
  |     change_events[] 追加
  |     meta.{source_root, scope, pst_root, coding_standards} 刷新
  |
  +-- pending_writebacks.json (Phase B 失败时):
        queue[]: {enqueued_at, source, lp_file, reason, payload}
```

### ELP 与 PSS 的间接关系

```
PSS 的产出如何影响 ELP:

  PSS 产出物                    ELP 如何使用
  +---------------------------+------------------------------------------+
  | LP 文件内容                | Phase A 的执行目标                        |
  | LP 中的 scope/allowed     | ELP 修改文件的边界约束                    |
  | LP 中的 Prereq/前置       | 依赖门 G.1 的前置来源之一                 |
  | README ## LP 序列         | 确定执行顺序 + 隐式前置 + 下一个 Prompt   |
  | README ## Coding Standards| ELP 修改代码时遵循的规范                  |
  | D 中的 AC (EARS)          | PST REVIEW 评估 ELP 执行质量的标准        |
  | Plan 架构描述              | PST REVIEW 评估 ELP 架构一致性的标准      |
  +---------------------------+------------------------------------------+

  注意: ELP 从不直接调用 PSS, PSS 也从不直接调用 ELP
  它们通过 PST (status.yaml) 和磁盘文件 (LP, README) 间接协作
```

### 状态映射规则

```
+------------+-----------------+-------------------+------------------------+
| ELP 结果   | 依赖门结果       | 映射 PST status   | 说明                    |
+------------+-----------------+-------------------+------------------------+
| completed  | passed          | ready             | 正常完成, HC 可用        |
| completed  | forced override | needs_update      | PST 不变量: ready需PC过 |
| completed  | verify-only     | needs_update      | verify 不能标 ready     |
| partial    | any             | needs_update      | 需要重新执行             |
| blocked    | any             | blocked           | 需要外部解决             |
+------------+-----------------+-------------------+------------------------+

白名单转换 (ELP 仅允许):
  draft -> ready
  ready -> ready
  ready -> needs_update
  ready -> blocked
  needs_update -> ready
  blocked -> ready

违反白名单时: 不写 transition, 仅追加 change_event 记录
```

## 功能

- 读取 LandingPrompt 目录 README.md 获取项目上下文（source_root、coding standards、LP 序列）
- 按步骤执行指定 LandingPrompt 中的实施任务
- 执行完成后自动同步状态到 `status/status.yaml`
- **Result 持久化** — Handoff Markdown 自动归档至 `pst_root/Result/`，保留完整执行历史
- 项目无关设计，支持任意 PST 管理的项目

## 调用方式

```text
Skill Execute-LandingPrompt + <path-to-landing-prompt>.md
```

## 前置条件

- LandingPrompt 目录必须包含 README.md（含 YAML front-matter 配置 `source_root`）
- 目标项目需遵循 project-state-tracker 工作流结构

## 安装

将本文件夹放置于 `~/.kiro/skills/Execute-LandingPrompt/`。

## 许可证

MIT
