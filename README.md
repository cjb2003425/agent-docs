# Hermes + Hindsight + clangd MCP 故障分析 SOP

> 适用场景：公司仅使用 Hermes；Hindsight 已启用 BGE-M3 与 reranker；C/C++ 源码位于 Ubuntu Server；已部署 [`felipeerias/clangd-mcp-server`](https://github.com/felipeerias/clangd-mcp-server)。

## 1. 目标

将已解决 Jira 的排障过程沉淀为可检索案例，并在分析新日志时：

1. 用 Hindsight 召回相似 Jira；
2. 用 clangd MCP 在**日志对应的源码版本与编译配置**中验证定义、引用、实现和调用链；
3. 用当前日志、历史案例和当前源码三类证据共同判断根因；
4. 输出带置信度、验证方法和源码依据的解决建议。

```text
Resolved Jira ──> Case Card ──> Hindsight bank: jira-incidents
                                      │
New log ──> jira-log-analysis Skill ─┼─> Similar Jira candidates
                                      │
                                      └─> clangd MCP ─> source evidence
                                                           │
                                                           └─> root cause / solution / verification
```

## 2. 职责边界

| 组件 | 职责 | 不应承担 |
|---|---|---|
| Hindsight | 历史 Jira、根因、方案、反证与验证经验的混合检索 | 判断当前分支是否仍存在同一代码路径 |
| clangd MCP | 当前 C/C++ workspace 的定义、引用、实现、类型与调用关系 | 直接认定历史 Jira 是当前根因 |
| Hermes | 编排检索、源码验证、证据比较与报告 | 无证据的确定性结论或生产改动 |
| WikiSkill（可选） | 从已审核诊断 trace 中优化 Skill 流程 | 运行时查询 Jira 或源码 |

历史案例是支持证据，当前日志与源码才是当前问题的事实来源。

## 3. 核心原则

1. **Case Card 先于 retain。** 不要将 Jira 原文、猜测性评论或大段日志直接写入 Hindsight。
2. **源代码版本必须对齐。** `Android14` 不足以定位源码；还必须确定产品、board、variant、manifest revision 和固定 commit。
3. **C/C++ 必须使用真实编译数据库。** `compile_commands.json` 应来自实际构建，包含正确的宏、include path、目标架构与工具链参数。
4. **稳定 signature 优先。** 异常、错误码、函数、服务、驱动、稳定栈帧和事件序列优先于时间戳/PID/地址。
5. **因果链优先于排名。** 排名第一的 Jira 若与当前事件序列或版本冲突，应被降级或排除。
6. **源码访问只读且可审计。** 不向 Agent 暴露 shell、写文件、构建、提交、推送或 sudo。
7. **Skill 更新必须评测。** 只有在 blind test 上证明更好且没有提高误导性修复率时，才更新生产 Skill。

## 4. 推荐目录与版本映射

### 4.1 Ubuntu Source Intelligence Server

```text
/srv/source-intelligence/
├── source/
│   ├── tv-a14-v1.24-userdebug/           # 固定源码版本，只读
│   └── tv-a14-v1.25-userdebug/
├── compile-db/
│   ├── tv-a14-v1.24-userdebug/compile_commands.json
│   └── tv-a14-v1.25-userdebug/compile_commands.json
├── clangd-cache/                         # service account 可写；源码目录不可写
├── source-map/
│   └── builds.yaml
└── bin/
    ├── clangd-mcp-tv-a14-v124
    └── clangd-mcp-tv-a14-v125
```

### 4.2 `builds.yaml`

每个日志 build 映射到唯一的 workspace；未知 build 不允许自动回退到主干。

```yaml
builds:
  TV-A-V1.24-userdebug:
    product: tv-a
    board: board-a
    variant: userdebug
    manifest_revision: "<manifest-commit>"
    source_root: /srv/source-intelligence/source/tv-a14-v1.24-userdebug
    compile_commands_dir: /srv/source-intelligence/compile-db/tv-a14-v1.24-userdebug
    source_commit: "<fixed-commit-or-manifest>"
    mcp_server: clangd_tv_a14_v124
```

Android 多仓代码应记录 manifest revision 与重要 repo commit。若日志没有 build fingerprint、产品或 variant，则源码结论只能是 Medium/Low confidence。

## 5. clangd MCP 服务配置

该项目通过 MCP stdio 调用 clangd LSP，要求 Node.js、clangd 与 `compile_commands.json`。它提供 `find_definition`、`find_references`、`find_implementations`、`get_call_hierarchy` 等工具。详见项目 [README](https://github.com/felipeerias/clangd-mcp-server)。

### 5.1 固定 workspace 启动脚本

每个受支持 build 使用固定脚本，避免 Agent 自行传入任意源码目录或 clangd 参数。

```bash
#!/usr/bin/env bash
# /srv/source-intelligence/bin/clangd-mcp-tv-a14-v124
set -euo pipefail

export PROJECT_ROOT=/srv/source-intelligence/source/tv-a14-v1.24-userdebug
export COMPILE_COMMANDS_DIR=/srv/source-intelligence/compile-db/tv-a14-v1.24-userdebug
export CLANGD_PATH=/opt/llvm/bin/clangd
export CLANGD_ARGS="--background-index --limit-results=1000"
export LOG_LEVEL=WARN
export CLANGD_LOG_LEVEL=error
export XDG_CACHE_HOME=/srv/source-intelligence/clangd-cache/tv-a14-v1.24-userdebug

exec /usr/local/bin/clangd-mcp-server
```

初期可以启用 `--background-index`。大型源码树若索引时间或内存不可接受，升级到 clangd Remote Index；该升级不改变 Hermes 的工具调用流程。参见 [clangd Remote Index](https://clangd.llvm.org/guides/remote-index)。

### 5.2 Hermes 通过 SSH 连接 stdio MCP

若 Hermes 与源码服务器不在同一主机，使用专用只读账号，通过 SSH 转发 MCP 的 stdin/stdout：

```yaml
mcp_servers:
  clangd_tv_a14_v124:
    command: ssh
    args:
      - -T
      - hermes-source-ro@source-server.company.internal
      - /srv/source-intelligence/bin/clangd-mcp-tv-a14-v124
    tools:
      include:
        - workspace_symbol_search
        - find_definition
        - find_references
        - find_implementations
        - get_call_hierarchy
        - get_type_hierarchy
        - get_hover
        - get_document_symbols
        - get_diagnostics
      resources: false
      prompts: false
```

工具名称以 MCP 启动后 Hermes 的 tool discovery 结果为准。只保留诊断所需工具。

### 5.3 权限与运维要求

- `hermes-source-ro` 无 sudo、无部署凭据、无生产服务权限。
- 源码 worktree 对该账号只读；仅 clangd cache 可写。
- 账号只可访问批准的 source roots 与启动脚本。
- SSH key、日志和命令调用接受公司审计。
- 将上游 `clangd-mcp-server` 固定到已验证 commit，并放入内部镜像；不要自动拉取 main。
- 该项目标注为 experimental，任何升级须在 sandbox 通过回归测试后再上线。

## 6. Hindsight 独立 Jira Bank

使用独立 `jira-incidents` bank，不要混入普通对话记忆、个人偏好或代码讨论。

建议 retain mission：

```text
Extract reusable troubleshooting knowledge from confirmed resolved Jira
incidents. Prioritize component, platform/version, symptom, stable log
signatures, event sequence, investigation, confirmed root cause, implemented
solution, verification, and negative evidence. Do not promote speculation or
rejected hypotheses to root cause.
```

建议分离环境：

```text
jira-incidents-prod  # 正常分析与批准后增量写入
jira-incidents-eval  # 固定快照，仅用于 blind test，禁止 retain
```

## 7. Jira Case Card Schema

```markdown
JIRA: <PROJECT-123>
TYPE: resolved_incident
STATUS: approved

TITLE: <one-line summary>

COMPONENT:
- <component>

PLATFORM: <Android/Linux/product; UNKNOWN if unavailable>
SOFTWARE_VERSION: <version/build/branch; UNKNOWN if unavailable>

SYMPTOM:
<Observed failure; do not state the conclusion here.>

LOG_SIGNATURES:
- <exception / error code / function / service / driver / stable frame>

EVENT_SEQUENCE:
1. <trigger>
2. <intermediate event>
3. <visible result>

INVESTIGATION:
- <reusable fact from the investigation>

ROOT_CAUSE:
<confirmed causal mechanism>

SOLUTION:
- <implemented code/config/process change>

VERIFICATION:
- <reproduction/regression step and pass criterion>

NEGATIVE_EVIDENCE:
- <ruled-out cause and why; NONE if unavailable>

CONFIDENCE: confirmed | probable
SOURCE_SCOPE: <applicable project / version range>
```

禁止写入时间戳、PID/TID、地址、UUID、随机 ID、整篇 Jira 评论、原始大日志、密钥或个人信息。

## 8. `jira-case-ingest` Skill

将以下内容存为 `~/.hermes/skills/debug/jira-case-ingest/SKILL.md`：

````markdown
---
name: jira-case-ingest
description: Extract reusable troubleshooting knowledge from resolved Jira incidents and retain approved Case Cards in the jira-incidents Hindsight bank.
---

# Jira Case Ingest

Use this skill only for resolved Jira issues with confirmed or strongly
supported root cause and a usable verification result.

1. Read title, description, relevant comments, logs, resolution, implemented
   change and verification evidence.
2. Separate confirmed facts, active hypotheses, rejected hypotheses, confirmed
   root cause, actual solution and verification.
3. Produce exactly one Case Card using the approved schema.
4. Preserve stable signatures; remove dynamic log fields and sensitive data.
5. Before retain, verify that the root cause is supported, the solution addresses
   it, negative evidence is correctly classified, and Jira ID is present.
6. If evidence is insufficient, save a draft for human review. Do not retain it
   as an approved resolved incident.
7. Retain the approved Card to `jira-incidents` with context
   `Resolved Jira incident <JIRA-KEY>`.
8. Run a post-retain recall using component + symptom + key signatures and
   report whether the new Card is discoverable.
````

## 9. `jira-log-analysis` Skill

将以下内容存为 `~/.hermes/skills/debug/jira-log-analysis/SKILL.md`：

````markdown
---
name: jira-log-analysis
description: Analyze C/C++ system logs using resolved Jira incidents in Hindsight and source validation through clangd MCP.
---

# Jira Log Analysis

Historical Jira is supporting evidence, not the answer. Current log and the
matching source workspace are required to confirm a source-level diagnosis.

## Phase 1 — Locate the failure

Identify failure time/window, symptom, component, process/thread, platform,
build fingerprint, product, board, variant, preceding event and relevant source
clues such as exception, error code, service, driver, function or stack frame.

## Phase 2 — Extract stable signatures

Use exception names, error codes, service/process/interface names, function
names, stable stack frames, drivers/subsystems, and state transitions. Ignore
timestamps, PID/TID, pointers, addresses, UUIDs and random counters.

Construct a 5–20 term recall query. Never submit the whole log to Hindsight.

## Phase 3 — Recall historical incidents

Search only `jira-incidents`. If the first result is weak, use at most three
queries: exact signature; component + symptom; trigger + event sequence.
Do not mutate queries until a desired answer appears.

## Phase 4 — Select source workspace

Map build fingerprint + product + board + variant to the approved clangd MCP
workspace. Record source root, compile database and fixed source commit.

If no exact mapping is available, do not claim a source-level root cause or
that a historical fix applies; report missing build/manifest evidence and
downgrade confidence.

## Phase 5 — Validate with clangd MCP

1. Use `workspace_symbol_search` to find source symbols from the log.
2. Use `find_definition` to locate the exact implementation.
3. Use `find_references`, `find_implementations`, and `get_call_hierarchy` to
   trace propagation, virtual dispatch, recovery, cleanup and retry paths.
4. Use `get_hover` or `get_type_hierarchy` when types or inheritance affect
   behavior.
5. Use `get_diagnostics` only for the relevant compilation unit.
6. Determine whether the historical Jira fix is present, absent, partial, or
   not comparable in this source revision.

## Phase 6 — Compare evidence

For the leading historical cases compare component, stable signatures, trigger
and event sequence, platform/version, symptom, negative evidence and current
source behavior. Prefer causal agreement over semantic similarity.

## Required output

## Problem
## Key Log Evidence
## Similar Historical Jira
## Evidence Comparison
## Source Validation
- build / product / variant
- source root / commit / compile database
- relevant files and symbols
- definitions, references or call-chain finding
- historical fix status
## Likely Root Cause
## Recommended Solution
Label the recommendation as confirmed / likely / hypothesis.
## Verification
## Confidence
## Missing Evidence

Never execute production changes. Never present a historical solution as
confirmed unless current log and matching source behavior support the same
causal mechanism.
````

## 10. 运行时查询流程

```text
new log
  -> failure window + stable signatures
  -> Hindsight recall: jira-incidents
  -> resolve build to clangd MCP workspace
  -> workspace_symbol_search
  -> find_definition
  -> find_references / find_implementations / get_call_hierarchy
  -> compare current log, source behavior and Jira Case Cards
  -> root cause + solution status + verification
```

典型 query：

```text
IOMMU fault foo_dma_map translation fault TV-A Android14 V1.24
```

不佳 query：

```text
设备异常，请分析
<粘贴 5 MB 原始日志>
```

## 11. Post-deployment Smoke Test

在一个已知源码 workspace 中验证：

1. `workspace_symbol_search` 找到一个已知函数。
2. `find_definition` 返回正确源码版本的定义位置。
3. `find_references` 返回关键调用方。
4. `get_call_hierarchy` 可展示一段预期调用关系。
5. `find_implementations` 可找到已知虚函数/接口实现。
6. `get_diagnostics` 不因错误的 compile database 产生系统性 include/macro 错误。

再选一个已关闭 Jira 和对应历史 log，验证 Agent 能正确：

```text
log -> similar Jira -> matching clangd workspace -> source evidence
    -> historical fix present / absent / partial -> confidence-calibrated result
```

## 12. Blind Test 与门禁

### 12.1 测试集

建立至少 20 个已知答案日志；测试提示不得包含 Jira Key、solution 或 commit。每条记录：

```csv
test_id,log_path,build_id,ground_truth_jira,ground_truth_root_cause,accepted_alternatives
BT-001,logs/BT-001.log,TV-A-V1.24-userdebug,TV-12345,stale DMA mapping,
```

### 12.2 指标

```text
Recall@5 = 正确 Jira 位于 Top 5 的样本数 / 总样本数
Root Cause Accuracy = 根因与 ground truth 一致的样本数 / 总样本数
Source Validation Success = 源码版本和关键符号定位正确的样本数 / 总样本数
Unsafe-solution Rate = 证据不足却给出确定修复的样本数 / 总样本数
```

### 12.3 失败分类

| 类型 | 优先调优方向 |
|---|---|
| 正确 Jira 未召回 | Case Card signature、同义组件名、query 构造 |
| 正确 Jira 排名过低 | 区分性特征、版本/事件序列、rerank 输入 |
| Jira 正确但根因错误 | 强化证据比较与反证规则 |
| 源码符号错误 | build map、compile database、clangd workspace |
| 方案误导 | 将版本差异、当前源码修复状态列为必填项 |

## 13. 增量维护

### 新 resolved Jira

```text
Jira close -> Case Card draft -> 技术 owner 复核 -> 脱敏/schema 检查
-> retain prod bank -> post-retain recall -> 更新台账
```

### 新源码版本

```text
build release -> 固定 manifest/commit -> 导出 compile_commands.json
-> 新建 clangd MCP 启动脚本 -> smoke test -> 增加 builds.yaml 映射
-> 允许 Hermes 使用该 workspace
```

### Skill 演进

将真实诊断 trace 脱敏后保存，定期总结 query miss、误召回、源码版本错配和有效验证步骤。可使用 WikiSkill 离线提出 Skill 改动，但必须在 `jira-incidents-eval` 固定快照与留出 blind test 上验证。未通过改动不得进入生产 Skill。

## 14. 常见失败模式

| 现象 | 原因 | 处理 |
|---|---|---|
| clangd 找不到定义 | build map 错误或 compilation database 不匹配 | 检查产品、variant、toolchain、`-I` 与 `-D` |
| 引用结果不完整 | 未完成索引或 compile database 缺 translation units | 修复数据库，重启/重建索引 |
| 宏分支与日志不一致 | 使用了错误产品/board 配置 | 严格匹配 build fingerprint 与 manifest |
| 相似 Jira 导致错误方案 | 只看 embedding/rank | 强制事件序列、源码路径和反证比较 |
| 主干已有 fix，旧版本没有 | 查错 workspace | 报告历史修复状态，切勿从主干推断旧版本已修复 |
| MCP 不稳定 | 实验性上游变动 | 固定 commit、内部镜像、smoke test、可回滚配置 |

## 15. 首期验收清单

- [ ] `jira-incidents-prod` 与 `jira-incidents-eval` 已隔离。
- [ ] 至少 30 个已审核 Case Card 已入库。
- [ ] 至少一个产品 build 已准确映射到源码、commit 和 compile database。
- [ ] clangd MCP 的九个工具中至少前五项 smoke test 通过。
- [ ] Hermes 仅能调用白名单 clangd MCP 工具。
- [ ] 至少 20 个 blind test 已建立并记录基线。
- [ ] 报告固定包含 build、commit、source validation 和 historical-fix status。
- [ ] 无源码映射时，Agent 能正确降级置信度而非猜测。

## 参考

- [clangd MCP Server](https://github.com/felipeerias/clangd-mcp-server)
- [clangd compilation database](https://clangd.llvm.org/design/compile-commands)
- [clangd remote index](https://clangd.llvm.org/guides/remote-index)
- [WikiSkill](https://github.com/ashutoshsinghpr7/wikiskill)
