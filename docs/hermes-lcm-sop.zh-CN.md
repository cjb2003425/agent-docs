# Hermes LCM 部署与长期记忆防丢 SOP

> 文档版本：1.0｜核验日期：2026-09-10  
> 适用场景：Hermes 长会话、代码分析、调试、个人知识管理。  
> 核验基线：第三方项目 `stephenschoettler/hermes-lcm`，源码提交 `8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54`，manifest 版本 `1.0.0-rc.1`。这是候选发布版本，不等于稳定版认证。  
> 本文已核对上游文档与源码，但未在用户的 Hermes 运行环境执行部署、压缩测试或恢复演练。推荐数值与验收目标属于本 SOP 的运维建议，不是实测结果或上游性能承诺。

## 目录

1. 背景与问题
2. LCM 原理与边界
3. 与其他记忆方案的关系
4. 安装前检查
5. Profile 与全局安装方案
6. 安装步骤
7. config.yaml 与环境变量
8. 推荐参数与调优
9. 验证方法
10. A/B 测试方案
11. 故障排查
12. 回滚方案
13. 升级与备份
14. 日常运维
15. 最佳实践
16. 验收清单
17. GitHub 发布与交接
18. 参考资料

## 1. 背景与问题

长会话超过模型可用上下文时，默认 compressor 会重建活动上下文。摘要可能保留“正在分析 Binder”，却省略设备路径、参数、错误原文、决策原因和后续更正。摘要再次被摘要时，这些遗漏可能累积。

必须区分三类问题：

| 现象 | 含义 | 首要处理 |
|---|---|---|
| 当前回答记不清 | 细节离开活动上下文 | 检索并展开历史原文 |
| 摘要缺失或错误 | 摘要模型、输入预算或摘要质量问题 | 检查模型路由、超时、日志与来源 |
| 原文无法恢复 | 未入库、过滤、数据库/外置文件丢失或损坏 | 检查存储边界、备份与恢复 |

**默认 compressor 不等于删除所有原文。** Hermes 的 session store 可保留历史，并通过 `session_search` 恢复。LCM 的价值在于将原文存储、摘要关系和回查工具组织成 context engine 的一部分。[S1][S4]

不能将此前对话中的 `compression.threshold: 0.75`、`protect_last_n: 40`、`target_ratio: 0.30` 直接视为 LCM 配置；也不能把这些值视为所有 Hermes 版本的最佳设置。当前宿主存在不同 tail 模式，具体字段作用应以安装版本为准。

目标是：活动上下文有界、入库历史可追溯、重要结论可持久维护、出现问题可恢复。提高上下文窗口或推迟压缩只能缓解部分症状。

## 2. LCM 原理与边界

LCM 是 Lossless Context Management。此实现先将消息写入 SQLite，再将旧消息组织为叶子摘要和更高层摘要；摘要节点保留来源关系，形成 DAG（有向无环图）。活动 prompt 使用摘要与近期原始消息，细节按需检索、分页展开。[S1][S2]

```text
新消息 ──→ 原始消息存储（SQLite / 必要时外置 payload）
                  │
                  └─→ 叶子摘要 ─→ 高层摘要 DAG
                                      │
系统提示 + 摘要前沿 + 最近消息 ────────→ 有界活动上下文
                                      │
                      搜索 → 查看节点 → 展开原文 → 核对答案
```

“Lossless”表示正常存储范围内保留恢复路径，**不表示模型始终看到全部原文，也不保证每次检索命中或推理正确**。摘要只是线索；精确路径、数值和引用必须回到原文验证。

边界包括：

- 启用前未进入 LCM 的旧会话，不会仅因启用插件自动变成完整的 LCM 历史。用 `session_search` 查旧记录，或采用上游明确支持的导入流程。
- ignore/stateless 规则会改变写入范围；敏感信息脱敏会有意移除原始敏感值。验收时必须说明这些例外。
- 外置 payload 是数据的一部分；只备份数据库可能无法恢复原始大输出或媒体内容。
- 数据库丢失、磁盘故障、人工清理和错误恢复不能靠 DAG 自行修复。
- SQLite/FTS 基线无需 embedding；未启用 embedding 时，不应承诺同义词或中文语义检索效果。

### 常用召回路径

| 需求 | 优先工具 | 后续动作 |
|---|---|---|
| 当前会话某个精确字符串 | `lcm_grep` | 对命中结果使用 `lcm_expand` |
| LCM 库内跨会话回忆 | `lcm_recall` | 检查范围、覆盖信息与原文引用 |
| 已知会话完整记录 | `lcm_load_session` | 按 `next_cursor` 分页 |
| 查看摘要结构 | `lcm_describe` | 再展开来源节点 |
| 按近期时间范围查摘要 | `lcm_recent` | 注意 UTC 时间边界与摘要覆盖 |
| 综合展开内容回答 | `lcm_expand_query` | 精确结论仍需核对原文 |
| 不在 LCM 库中的旧会话 | Hermes `session_search` | 保存命中的 session 和来源 |

工具是 Agent 工具，不是可直接在终端执行的命令；具体参数让 Agent 按当前工具 schema 填写。新版 `lcm_grep` 也支持显式扩大 session 范围，但不同范围返回类型不同。[S1]

## 3. 与其他记忆方案的关系

| 组件 | 主要解决的问题 | 不能替代什么 | 与 LCM 的搭配 |
|---|---|---|---|
| 默认 compressor | 控制活动上下文长度 | 不能保证摘要保留全部细节 | 同一个 context engine 槽位选择其一 |
| Hermes MEMORY.md / USER.md | 少量稳定事实与偏好 | 不适合原始日志和大量证据 | 存放高价值结论和索引 |
| Hindsight | 跨会话记忆、实体关系、检索与反思 | 不能自动证明每次回答都有原文依据 | 独立 memory provider，按需与 LCM 配合 |
| LanceDB | 向量/混合检索存储基础设施 | 数据库本身不是完整记忆策略 | 通过专用插件接入，或用于其他知识库 |
| `lancedb/hermes-agent-memory` | 提取持久事实并召回，带来源记录 | 不负责取代 LCM 的摘要 DAG | 作为 memory provider 候选 |
| LLM Wiki | 将资料编译为可维护、可审阅的 Markdown 知识库 | 不自动保留每一轮会话 | 收纳经过核验的知识、决策、操作手册 |

Hermes 的 `context.engine` 与 `memory.provider` 是不同配置槽位。可以设计为 LCM 管理活动上下文，Hindsight **或** LanceDB 插件管理长期记忆；不要假设在单选 memory provider 槽位中能同时填写两个 provider。架构上可组合不代表已在用户环境验证所有生命周期 hook。[S4][S5][S6]

Hindsight 提供 retain/recall/reflect，支持云端或本地方案；本地运行也可能调用外部 LLM。LanceDB 的 Hermes 插件以事实召回为主，当前文档描述默认使用外部 embedding API；“本地数据库”不等于“所有处理离线”。[S5][S6]

LLM Wiki 是知识维护模式，不是 Hermes 的标准安装依赖，也不必先安装一个同名包。建议保留 sources、主题页、决策页、索引与变更记录；每个结论能追溯来源并标记更新时间。[S7]

**推荐实施顺序：LCM 基线 → 备份与召回验收 → Wiki/MEMORY 规则 → 再评估一个外部 memory provider。** 避免一次开启多个自动记忆系统而无法判断收益与额外开销。

## 4. 安装前检查

### 4.1 记录环境

在实际运行 Hermes 的机器、容器或 WSL 环境中执行。本文终端安装示例使用 Bash，适用于 Linux/macOS/WSL；不要直接粘贴进 PowerShell。Windows 原生部署需要另行核对当前安装器与符号链接支持，不能假定 Bash 脚本原生可运行。

```bash
hermes --version
hermes profile
hermes profile --help
hermes plugins
git --version
```

记录：宿主版本/提交、主模型和有效窗口、auxiliary compression 模型、启动方式、实际 profile home、数据库位置、插件版本、磁盘容量。命令若不存在，先查当前版本 `--help`，不要据此猜配置生效。

上游基线依赖契约声明 Hermes `>=0.16,<1`、Python 3.11–3.14，并需要 context engine 插件接口。这是本次源码基线的范围，不代表未来版本承诺。使用 **Hermes 实际解释器** 检查 Python 与 SQLite FTS5，而非随手调用系统 Python。[S3]

```bash
# 替换为 Hermes 实际使用的解释器绝对路径
HERMES_PY='/absolute/path/to/hermes/venv/bin/python'
"$HERMES_PY" --version
"$HERMES_PY" -c 'import sqlite3; c=sqlite3.connect(":memory:"); c.execute("CREATE VIRTUAL TABLE t USING fts5(x)"); print("SQLite", sqlite3.sqlite_version, "FTS5 OK")'
```

### 4.2 开始前门禁

- 确认目标 profile 与配置文件路径；检查 sticky default，命令尽量显式带 `-p`。
- 确认主模型有效窗口真实，不能靠虚报 context length 获得更多容量。
- 验证辅助模型能访问、预算足够、支持实际摘要输入规模；LCM 并非始终只传固定 20K 输入。
- 保存原 `config.yaml`、`.env`、启动服务配置与版本号；已有数据按第 13 节一致性备份。
- 确認存储空间能容纳库、WAL、payload、备份和临时恢复副本；建议至少保留当前数据集数倍空间。
- 同一个 profile 不运行两个独立 Agent 写入进程；独立 Agent 使用独立 profile。[S8]
- 不将凭据、数据库、原始会话或日志提交到公开 Git 仓库。

## 5. Profile 与全局安装方案

这里“全局”指当前用户 Hermes 根目录的插件安装，不是系统级 Python 安装，也不意味着所有 profile 共用记忆。

| 项目 | 默认用户目录 | 独立 profile（推荐先试点） |
|---|---|---|
| 示例 home | `~/.hermes` | `~/.hermes/profiles/lcm-pilot` |
| 插件 | `<home>/plugins/hermes-lcm` | 同左，在 profile 内 |
| Skill | `<home>/skills/hermes-lcm` | 同左，在 profile 内 |
| 配置 | `<home>/config.yaml`、`.env` | 同左，在 profile 内 |
| 默认 LCM DB | `<运行时 HERMES_HOME>/lcm.db` | 应落在该 profile home |
| 启动 | `hermes -p default chat` | `hermes -p lcm-pilot chat` |

若设置了 `LCM_DATABASE_PATH`，数据库可能在别处；以 `lcm_status` 为准。不要仅靠路径表推断隔离成功。

插件安装脚本将 `HERMES_HOME` 当作基础目录，再追加 `profiles/$HERMES_PROFILE`。**安装时不要同时把 `HERMES_HOME` 指向某个 profile，并再次设置 `HERMES_PROFILE`，否则会形成嵌套路径。** 运行时的 `HERMES_HOME` 则应是宿主解析后的实际 profile home。[S2][S9]

## 6. 安装步骤

### 6.1 推荐：独立 profile 试点

先创建空 profile，配置主模型与辅助模型；不要复制生产 bot token 后同时启动两个 gateway。

```bash
hermes profile create lcm-pilot
hermes -p lcm-pilot setup
hermes -p lcm-pilot chat
```

确认普通对话正常后退出。以下命令假定基础目录确实是 `$HOME/.hermes`；自定义目录需替换 `HERMES_BASE`。

```bash
HERMES_BASE="$HOME/.hermes"
LCM_DIR="$HERMES_BASE/profiles/lcm-pilot/plugins/hermes-lcm"
git clone https://github.com/stephenschoettler/hermes-lcm.git "$LCM_DIR"
git -C "$LCM_DIR" checkout --detach 8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54
cd "$LCM_DIR"
HERMES_HOME="$HERMES_BASE" HERMES_PROFILE=lcm-pilot bash scripts/install.sh
```

固定提交用于复现本 SOP，不代表应无限期使用候选版本。生产部署先在独立环境验证选定 release；换版本时重查参数与迁移说明。

### 6.2 备选：默认用户目录安装

只执行一种安装方案，避免无意装入多个位置。

```bash
HERMES_BASE="$HOME/.hermes"
LCM_DIR="$HERMES_BASE/plugins/hermes-lcm"
git clone https://github.com/stephenschoettler/hermes-lcm.git "$LCM_DIR"
git -C "$LCM_DIR" checkout --detach 8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54
cd "$LCM_DIR"
HERMES_HOME="$HERMES_BASE" HERMES_PROFILE= bash scripts/install.sh
```

已有 checkout 时先检查 `git status --short` 和 remote，不要覆盖目录或强制重置修改。安装脚本遇到插件或 skill 路径冲突会拒绝执行；先核对冲突来源，备份后再处理。

即使 clone 位置已经正确，也运行 `scripts/install.sh`，让随附召回 skill 出现在对应 profile 的 skills 目录。基础运行不需要 `pip install hermes-lcm`；可选功能依赖按所选版本说明安装到宿主环境。[S1][S3]

可选的依赖契约核验：

```bash
"$HERMES_PY" scripts/validate_dependency_contract.py --report-environment
```

这不替代实际宿主运行测试，也不等于安全漏洞扫描。

## 7. config.yaml 与环境变量

### 7.1 最小激活配置

编辑目标 profile 的 `config.yaml`，**合并到已有键中**，保留原来的插件列表、模型和 provider 配置，不要整文件覆盖或重复顶层键。

```yaml
plugins:
  enabled:
    - hermes-lcm
    # 在此保留已有的其他插件

context:
  engine: lcm

compression:
  enabled: true
```

插件名是 `hermes-lcm`，引擎名是 `lcm`。只写一个可能无法激活。`compression.enabled` 仍是宿主允许压缩的总开关，使用 LCM 时保持开启。[S2]

### 7.2 推荐试点环境变量

将以下非密钥设置合并到该 profile 的 `.env`；已有同名项时替换，避免重复定义。若 gateway 使用服务或容器启动，确认它读到该文件或相同环境变量。只在交互终端 `export` 不保证服务进程继承。

```dotenv
LCM_CONTEXT_THRESHOLD=0.35
LCM_FRESH_TAIL_COUNT=32
LCM_FRESH_TAIL_MAX_TOKENS=0
LCM_LEAF_CHUNK_TOKENS=20000
LCM_INCREMENTAL_MAX_DEPTH=3
LCM_EMBEDDINGS_ENABLED=false
LCM_TEMPORAL_ROLLUPS_ENABLED=false
LCM_LARGE_OUTPUT_EXTERNALIZATION_ENABLED=false
LCM_LARGE_OUTPUT_TRANSCRIPT_GC_ENABLED=false
LCM_PROACTIVE_RECALL_ENABLED=false
LCM_DOCTOR_CLEAN_APPLY_ENABLED=false
```

这里显式固定阈值，是为了减少继承既有配置带来的歧义。无需首次部署就设置模型覆盖、数据库自定义路径、语义检索和自动注入。

仅可信操作终端需要 `/lcm` 管理命令时，再加入：

```dotenv
LCM_ENABLE_SLASH_COMMAND=true
```

不开启也能使用 Agent 的 `lcm_status` / `lcm_doctor` 工具；`/lcm` 并非默认可用。

### 7.3 当前源码中的阈值优先级

本次源码 `config.py` 的加载优先级是：

```text
有效 LCM_CONTEXT_THRESHOLD 环境变量
  > config.yaml 的 lcm.context_threshold
  > config.yaml 的 compression.threshold（compression 未关闭时）
  > 内置回退值 0.35
```

源码基线仅把 `lcm.context_threshold` 列为受支持的 `lcm:` YAML 字段，其余字段应使用 `LCM_*`。因此不能编造 `lcm.fresh_tail_count`、`lcm.database_path` 等 YAML 配置。[S10]

若希望只用 YAML 管理阈值，可删除 `.env` 和服务环境中的同名覆盖后加入：

```yaml
lcm:
  context_threshold: 0.35
```

上游操作指南的“阈值归属”叙述比当前源码更简略；本文明确保留这一差别。部分模型路由还存在运行时预算调整，**最终读取普通消息初始化后的 `lcm_status`，核对有效窗口、threshold tokens、配置来源和警告**，不能只看配置文件或启动 banner。

### 7.4 重启并检查

CLI：退出原会话进程后重新启动目标 profile。Gateway：通过原有服务管理方式重启；支持当前 CLI 的部署可执行：

```bash
hermes -p lcm-pilot gateway restart
```

不要为了套用示例另起一个重复进程。插件、引擎、环境变量修改统一按完整重启处理；不要套用默认 compressor 某些字段的热加载行为。

## 8. 推荐参数与调优

以下默认值按核验源码/文档列示。推荐值是起点，需要用第 10 节测量。

| 参数 | 基线默认 | 初始建议与取舍 |
|---|---|---|
| `LCM_CONTEXT_THRESHOLD` | 回退 0.35，可继承 | 显式 0.35；验证后逐步提高，降低压缩次数但增加 prompt 成本 |
| `LCM_FRESH_TAIL_COUNT` | 32 条消息 | 32；工具密集时关注实际 token，不把消息条数当用户轮数 |
| `LCM_FRESH_TAIL_MAX_TOKENS` | 0，无上限 | 先 0；tail 过大可试 8000–16000，仅为调优候选 |
| `LCM_LEAF_CHUNK_TOKENS` | 20000 | 先默认；小窗口/摘要输入过大时结合 dynamic chunking 验证 |
| `LCM_INCREMENTAL_MAX_DEPTH` | 3 | 保持 3；不要一开始无限层级 |
| `LCM_DYNAMIC_LEAF_CHUNK_ENABLED` | false | 基线关闭；需要限制单次处理规模时单独试验 |
| `LCM_SUMMARY_MODEL` | 空，宿主辅助模型 | 使用已验证路由，不填写虚构模型 ID |
| `LCM_SUMMARY_TIMEOUT_MS` | 回退 60000，可继承宿主 timeout | 慢模型实测后可试 120000，单位毫秒 |
| `LCM_EXPANSION_CONTEXT_TOKENS` | 32000 | 不超过辅助模型可容纳的实际输入预算 |
| `LCM_SUMMARY_SPEND_MAX_CALLS` | 24 | 保持费用保护，排查反复失败后再改 |
| `LCM_DATABASE_PATH` | 运行时 home 下 lcm.db | 初期不覆盖；必要时显式绝对路径且各 profile 独立 |

尾部 token 上限也不是绝对硬截断：实现保留最新消息及完整 assistant/tool 分组，超大单条消息仍可能带来压力。[S2]

预算估算：`触发 token ≈ 有效窗口 × 阈值`。128K 窗口下 0.35 约为 44.8K；1M 窗口下 0.75 约为 750K。还需给系统提示、工具 schema、输出、下一轮大工具结果留空间，不能把触发值当作总费用或安全上限。

调参流程：先采集至少 3 次压缩 → 找到延迟、成本或召回瓶颈 → 每次只改一个参数 → 重启 → 重复同一测试集。摘要失败时先查实际输入与路由；默认 `LEAF_CHUNK_TOKENS` 不是无条件的单次输入硬上限。

进阶开关逐个评估：embedding 提升语义检索但引入模型/费用/索引维护；temporal rollups 提供时间摘要但增加派生处理；大输出外置需连同 payload 备份；主动召回会增加注入信息和误匹配机会。不要把 benchmark 中不存在于运行时的字段直接写入配置。[S2]

## 9. 验证方法

### 9.1 加载、绑定与存储验证

```bash
hermes -p lcm-pilot plugins
hermes -p lcm-pilot chat
```

发送一条普通消息，例如“这是 LCM 部署验收会话”。然后请 Agent 调用 `lcm_status`、`lcm_inspect`、`lcm_doctor`。管理命令已启用时也可使用：

```text
/lcm status
/lcm doctor
```

记录：插件路径和 git commit、有效引擎、session/conversation ID、实际 DB 路径、配置来源、有效窗口与阈值、压缩次数、读写/FTS 健康、外置文件状态。刚重启未发普通消息时出现 `unbound` 或 `uninitialized`，不立即判失败。

基线 manifest 暴露 15 个工具 schema；只看到工具名称不等于对应 opt-in 功能已启用。最少实际调用状态、诊断、搜索和展开工具，确认有效响应。

### 9.2 压缩后原文恢复测试

使用合成资料，不使用真实凭据。把以下事实分散放入会话前段，并夹入足够的不同调试内容，使真实上下文达到触发条件：

```text
验收标识：LCM-SOP-TEST-7429
设备路径：/dev/vndbinder
构建命令：ninja -C out/target demo_service
初始重启间隔：17 秒
后续确认：重启间隔改为 23 秒，17 秒方案已废弃
保留旧接口的原因：供应商客户端仍依赖旧协议
禁止项：未经确认不得删除旧兼容层
中文事实：联系人代号为“青竹”，负责验证重连顺序
```

1. 保留离线答案表及原始输入，避免生成答案后反过来充当真值。
2. 等待至少一次**状态可证实的压缩**。仅发送“请记住”不算压缩测试。
3. 若测试窗口过大，可在试点临时降低阈值、tail 和 chunk 设置帮助触发，记录改动；该结果只用于机制验证，不代表生产性能。
4. 问设备路径、构建命令、最终间隔、废弃值和原因，要求返回对应原文引用及 session/store/node 标识。
5. 对每个命中实际 `lcm_expand`，逐字核对路径、命令、17/23 及更正关系。
6. 连续经历至少 3 次压缩后重复询问，确认未把旧值当现值。
7. 重启 Hermes 后复测；另建会话，用 `lcm_recall` 或指定 session 读取旧事实。
8. 用一段已确认发生外置的大输出做展开测试；未发生外置则记“不适用”，不能记通过。

`/lcm rotate apply` 不调用摘要模型，不能替代“实际摘要压缩发生”的证明。它会推进活动前沿，原文仍可恢复；若需要摘要覆盖，应先正常压缩。[S2]

### 9.3 隔离与恢复测试

建立第二个测试 profile，放入不同 canary，核对实际 DB 路径不同；第二 profile 不应通过本库召回第一 profile 的记录。外部 memory provider 的共享 bank/workspace 必须另测，profile 不是文件系统沙箱。

在单独恢复目录还原备份，用固定版本进行 doctor 与同一 canary 回查，确认数据库和外置 payload 都可读。完整步骤见第 13 节。

## 10. A/B 测试方案

### 10.1 实验分组

| 组别 | Context engine | 长期记忆条件 | 目的 |
|---|---|---|---|
| A | 默认 compressor | 固定 MEMORY/USER；允许 session_search | 与真实可恢复基线公平比较 |
| B | LCM | 与 A 相同；允许 LCM 工具 | 测量 LCM 增益 |
| C（可选） | LCM | 增加一个 memory provider 或 Wiki 流程 | 单独测量组合增益 |

使用互相隔离且干净的 profile/数据库，保持模型、供应商、有效窗口、任务内容、工具权限、生成设置相同。主实验尽量匹配触发预算，比较引擎；另设“各自推荐参数”实验比较整套体验，不混合解释结论。

相同脚本重放至少 3 次，执行顺序交替。禁止 A/B 共用历史库或答案文件；答案文件只给评分者。除了允许检索的实用测试，可另做不使用工具的闭卷测试，但分别报告。

### 10.2 数据集与指标

建议准备 30 个问题：精确路径/命令 8、数值与更正 6、决策原因 6、中文/同义表达 4、跨会话 4、历史不存在的问题 2。确保相关事实已离开 raw tail，并经过至少 3 次压缩。

| 指标 | 计算或记录方法 |
|---|---|
| 精确正确率 | 逐字/结构核对正确题数 ÷ 对应题数 |
| 来源有效率 | 展开后确实支持结论的引用数 ÷ 全部引用数 |
| 更正处理正确率 | 正确采用现值并说明旧值失效的题数 ÷ 更正题数 |
| 幻觉率 | 捏造事实或来源的题数 ÷ 总题数 |
| 恢复覆盖 | 可展开的抽样消息/外置对象数 ÷ 应保留样本数 |
| 性能 | 每题端到端耗时、检索耗时、p50/p95、失败次数 |
| 成本 | 主模型、摘要、embedding、展开综合 token 与实际账单 |
| 存储 | DB、WAL、payload、备份大小及增长量 |

建议试点通过门槛：关键路径/命令/更正题 100% 正确；总体精确正确率至少 95%；已给出的证据 100% 可验证；不存在问题不捏造；抽样原文恢复 100%；备份恢复通过。延迟和费用阈值由实际预算事先填写。小样本只表示验收结果，不可外推全场景保证。

记录表模板：

```csv
run_id,group,question_id,model,host_commit,lcm_commit,config_hash,compactions,correct,evidence_valid,latency_ms,cost,notes
```

出现一次关键事实错答，区分“搜索不到、展开不全、原文缺失、答案推理错误”后修复并重跑受影响案例。

## 11. 故障排查

| 症状 | 先查什么 | 处理方式 |
|---|---|---|
| 插件找不到 | 加载路径、manifest、enabled 列表、宿主兼容性 | 校正 profile，合并配置，重启 |
| `lcm (not found)` 但工具存在 | 普通消息后 status 是否绑定并响应 | 可能为宿主发现/展示不一致，不立即删库重装 |
| 日志出现 `Path B fallback` | 旧宿主的 context-engine 工具路径 | 上游说明可能正常，以实际工具调用验证 |
| `/lcm` 不识别 | slash 开关和进程环境 | 可信终端启用后重启，或调用 Agent 工具 |
| status 未绑定 | 是否已发普通消息 | 初始化后再查 |
| 改参数无效 | 改错 profile、服务继承旧 env、重复变量、YAML 无效字段 | 查 `config_sources` 与 ignored-key 警告，重启 |
| 不触发压缩 | enabled 总开关、有效阈值、tail、backlog 和 skip 原因 | 看真实 token 与压缩状态，不仅看对话轮数 |
| 摘要失败/反复压缩 | 429/401/超时、辅助窗口、费用保护、熔断状态 | 修复路由/预算，检查原文可恢复后重试 |
| 搜索不到旧事实 | 是否入库、session 范围、过滤、中文分词、页游标 | 从短精确词检索，扩大正确范围，分页展开 |
| 英文可查中文难查 | FTS 词法限制、没有 semantic embedding | 精确标识检索作为基线；单独验证中文 embedding |
| 回答采用旧结论 | 新旧原文和来源时间 | 展开更正记录，明确有效时间和废弃原因 |
| DB locked | 多进程写同一 home、文件系统、长事务 | 停重复 writer，保留库与日志，避免活跃时搬库 |
| FTS/SQLite 异常 | doctor、磁盘、权限和一致性 | 先备份，再只读 repair 诊断；需要时恢复已知好备份 |
| 外置内容失联 | payload 路径、权限、备份是否包含、文件是否迁移 | 恢复配套目录，保留相对关系和 ref |
| 库增长快 | 大输出、WAL、外置 payload、备份轮换 | 减少噪声采集，规划保留期，不直接删消息表 |
| 跨 profile 串库 | LCM_DATABASE_PATH、服务环境、外部 provider bank | 恢复各自路径与命名空间后重新验收 |

诊断从 `lcm_status` → `lcm_inspect` → `lcm_doctor` 开始。启用管理命令时可先执行 `/lcm doctor repair` 的只读诊断；`apply`、clean、rotate 都是另外的状态变更，先看预览与备份，再由操作员按实际问题决定。[S2]

不要把“提高 timeout、增大上下文、关闭所有限制、删除 lcm.db”作为通用修复步骤。

## 12. 回滚方案

### 12.1 快速切回默认 compressor

1. 记录故障时间、插件提交与 DB 路径；保留日志和当前数据快照。
2. 停止目标 profile 的所有写入进程。
3. 在原配置中将引擎改回：

```yaml
context:
  engine: compressor
compression:
  enabled: true
```

4. 从 `plugins.enabled` 中只移除 `hermes-lcm`，保留其他插件；恢复安装前 compressor 参数。
5. 重启同一 profile，用普通消息确认默认引擎工作，再验证 `session_search`。
6. 保留 LCM 数据库、payload、插件与备份。切回 compressor **不会自动把 LCM DAG 导入默认摘要**，LCM 独有数据需要日后用兼容版本恢复查看。

不要通过删除数据库来“关闭 LCM”。

### 12.2 回退插件版本

停止所有 writer → 保存故障后的完整快照 → checkout 安装前已记录的好版本 → 恢复该版本对应的升级前备份与配置 → 重启 → doctor 和 canary 验证。

上游针对本候选版本明确要求：回退旧版时恢复升级前备份，不让旧版直接打开新版修改过的数据库。升级后新增会话要保留在故障快照中，不能假设能自动合并进旧库。[S2]

## 13. 升级与备份

### 13.1 备份范围

| 数据 | 为什么需要 |
|---|---|
| LCM DB | 原始消息、DAG、检索/派生状态 |
| 配套外置 payload 目录 | 大文本与媒体原始内容可能不在 DB 内 |
| Hermes state.db 和实际会话存储 | 宿主历史与跨会话恢复 |
| MEMORY/USER、Wiki | 已整理的长期事实与决策 |
| config.yaml、.env、服务配置 | 还原同一运行行为；含密钥的备份加密、限权 |
| 宿主/插件提交、环境清单 | 数据格式与行为可复现 |

不要把 `hermes profile create --clone-all` 当作已验证的完整历史备份；当前 profile 文档明确指出部分历史等内容被排除。[S8]

### 13.2 在线与停机备份

**上游支持的 LCM 在线 DB 备份路径是 `/lcm backup`。** 记录返回路径和时间，随后在副本上检查。它不是整个 profile 或外置 payload 的完整备份。[S2]

完整灾备优先采用短暂停机：

1. 停止该 profile 的 gateway、CLI 和所有其他 writer，确认服务不会自动拉起。
2. 核对 status 之前报告的 DB 与外置路径；若自定义到 profile 外，单独纳入清单。
3. 将 `lcm.db` 及仍存在的 `lcm.db-wal`、`lcm.db-shm` 作为一个静止快照保存；不要在活跃写入时逐个复制。
4. 同时保存 payload、宿主历史、配置、记忆文件和版本清单。
5. 在备份副本执行 SQLite `PRAGMA quick_check`，预期结果为 `ok`；记录文件数量、大小和校验和。
6. 恢复服务并发一条普通消息，确认健康。

保留建议：每天一次（7 份）、每周一次（4 份）、每月一次（3 份），升级前额外一份；根据会话敏感性、容量与可接受损失调整。建议日常 RPO ≤24 小时，关键变更前额外备份；RTO 目标先定为 60 分钟并通过演练验证。

### 13.3 恢复演练

1. 新建隔离恢复目录/测试 profile，禁用 bot、cron 与其他外部发送入口。
2. 使用相同宿主与插件提交，恢复 DB、payload、配置结构；API 凭据按需单独配置。
3. 从 online backup 恢复时，将 SQLite 快照恢复为目标 DB，不能混入旧目录的 WAL/SHM；停机整套恢复则保持同一快照配套文件。
4. 若路径变化，检查外置 ref 的解析与 `LCM_DATABASE_PATH`；不要直接修改生产数据表修路径。
5. 先 SQLite 检查，再运行 doctor；抽查压缩前事实、更正记录、跨会话和外置大输出。
6. 记录耗时、失败项和恢复范围；全部通过才将该备份标为可恢复。

仅看到备份文件存在，不算恢复成功。

### 13.4 升级流程

1. 读目标 release 的 CHANGELOG、依赖契约、schema 和回滚说明。
2. 在试点复制生产备份进行验证，记录旧提交；检查 checkout 无未保存修改。
3. 完整备份后停止生产 writer，再更新到已验证 tag/commit。

```bash
git -C "$LCM_DIR" status --short
git -C "$LCM_DIR" rev-parse HEAD
git -C "$LCM_DIR" fetch --tags origin
# 将下列值替换为本次已验证的实际 tag 或 commit
LCM_TARGET_REF='REPLACE_WITH_VERIFIED_TAG_OR_COMMIT'
git -C "$LCM_DIR" checkout --detach "$LCM_TARGET_REF"
```

4. 必要时重跑相同 profile 的 installer 以检查 skill 链接；重启并发普通消息。
5. 核对 loaded commit、DB 路径、doctor、canary 和摘要/展开，观察一次真实压缩。
6. 失败则按第 12 节回滚，保留升级后快照。

本基线的 core schema 为 5；上游明确说明从 v0.20.0/v0.21.0-rc2 到该 RC 不需要手工核心迁移，新增可选数据按功能创建。这个结论不能推广到任意未来升级。[S2]

固定提交的部署不要直接使用 `git pull`；仅跟踪分支且工作树干净时才按上游使用 `git pull --ff-only`，生产更推荐升级到明确验证过的版本。

## 14. 日常运维

| 频率 | 操作 | 留存结果 |
|---|---|---|
| 每次长任务开始 | 确认 profile、目标、最近更正与知识索引 | 当前任务状态 |
| 每个关键结论确认后 | 写入 Wiki/适当记忆文件，附来源与有效时间 | 可追溯结论 |
| 每日 | 检查错误、容量、备份成功与摘要开销 | 异常清单和备份时间 |
| 每周 | doctor、固定 canary 回查、抽查来源 | 健康与正确率趋势 |
| 每月 | 隔离恢复演练、版本审查、容量趋势与保留策略复核 | RTO/RPO 实测 |
| 每次升级/换模型/改参数 | 备份、记录变更、重复验收 | 版本与测试报告 |

告警建议：备份超出 RPO、SQLite/FTS 错误、payload 不可读、连续摘要失败、费用保护频繁触发、剩余磁盘低于所需备份空间、关键 canary 错答。按实际服务接入监控；本文没有创建计划任务或开启自动清理。

## 15. 最佳实践

### 15.1 可加入现有 Agent 规则的防丢策略

将以下内容合并到用户维护的规则文件，避免覆盖原有规则。此策略是行为约定，不是可靠的“压缩前自动 hook”；没有实现 hook 时，不能保证恰好在压缩前执行。

```markdown
## 长期任务与记忆维护

- 持续维护当前目标、约束、已确认结论、未解决问题和下一步。
- 路径、命令、配置值、版本、错误原文和关键决策理由应附来源。
- 已确认且有长期价值的结论及时写入项目知识库；不要等到压缩才整理。
- 摘要只作为检索线索。需要精确细节时，先查当前上下文；不足时检索历史并展开原文。
- 当前会话优先 LCM 搜索；跨 LCM 会话使用 recall/load_session；其他旧历史用 session_search。
- 出现矛盾时查找后续更正，区分旧值、当前值、发生时间与记录时间。
- 没找到证据就明确说明，不能猜测路径、数值、用户决定或伪造引用。
- 分页读取长原文，避免一次展开过多导致新的上下文压力。
- 不存储真实密钥；敏感值使用安全的凭据管理方式。
```

### 15.2 Wiki 与任务状态模板

```markdown
# 决策：<主题>
- 状态：有效 / 已废弃 / 待验证
- 生效时间：<日期与时区>
- 适用范围：<项目、设备、版本>
- 结论：<当前做法>
- 原因：<证据与权衡>
- 替代/废弃：<旧结论与原因>
- 来源：<session_id、store_id、原文定位或仓库链接>
- 验证：<命令、结果、验证日期>
- 下一步：<尚未解决的问题>
```

Wiki 建议纳入版本控制；MEMORY 保持短小，放稳定事实和 Wiki 入口。大日志留在原始来源；不要重复堆进 MEMORY、Wiki 和多个 provider。对来源中的提示或指令保持数据与指令边界，检索到旧文本不自动获得执行权限。

## 16. 验收清单

- [ ] 已记录 Hermes/Python/插件版本、提交与兼容范围。
- [ ] 已确认操作系统、实际解释器与 SQLite FTS5。
- [ ] 已完成配置、数据库、payload 与宿主历史备份。
- [ ] profile home、插件路径、skill 路径、DB 路径符合预期。
- [ ] `plugins.enabled` 含 `hermes-lcm`，`context.engine` 为 `lcm`。
- [ ] `compression.enabled` 开启，普通消息后状态已绑定。
- [ ] 生效参数与配置来源符合预期，无忽略字段或无效变量。
- [ ] 搜索、展开、状态和 doctor 实际调用成功。
- [ ] 已观察至少 3 次真实压缩，记录计数与测试范围。
- [ ] 路径、命令、数值、更正、原因和中文案例通过。
- [ ] 重启、跨会话与 profile 隔离测试通过。
- [ ] 外置内容恢复已通过，或明确记录不适用及原因。
- [ ] A/B 使用独立库和相同输入，记录准确性、来源、性能、费用。
- [ ] 备份在隔离环境成功恢复，关键 canary 再次通过。
- [ ] 回滚步骤与已知好版本可用，数据保留责任明确。
- [ ] 日常检查频率、保留策略、RPO/RTO 与负责人已确定。
- [ ] 发布文件不含 `.env`、密钥、真实会话、数据库或私有日志。

验收签署：操作者 ______；日期 ______；环境 ______；结果（通过/有条件通过/失败）______；例外与后续事项 ______。

## 17. GitHub 发布与交接

建议仓库路径：`docs/hermes-lcm-sop.zh-CN.md`。只上传本 Markdown；研究用源码副本和运行数据不属于交付内容。

发布仓库：[`cjb2003425/hermes-jira-log-analysis-sop`](https://github.com/cjb2003425/hermes-jira-log-analysis-sop)。该仓库创建于 2026-09-09，由用户指定用于保存本文。

目标分支：`main`；文件路径：`docs/hermes-lcm-sop.zh-CN.md`。后续维护采用该文件路径；如启用分支保护，则通过文档分支和 PR 更新。无需在聊天里提供 token。

上传前读取同名文件，避免覆盖无关内容；上传后通过 GitHub 重新读取文件核对内容，返回文件链接与提交或 PR 链接。若账号无写权限，需要连接有权限的账号或授予仓库写入权限。

## 18. 参考资料

优先以所选版本源码和运行时状态为准。以下链接中的固定提交用于复现；官方网站及其他项目链接会随时间更新。

- [S1：hermes-lcm README（固定提交）](https://github.com/stephenschoettler/hermes-lcm/blob/8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54/README.md)
- [S2：hermes-lcm Operator Guide（固定提交）](https://github.com/stephenschoettler/hermes-lcm/blob/8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54/docs/operator-guide.md)
- [S3：依赖契约说明](https://github.com/stephenschoettler/hermes-lcm/blob/8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54/docs/dependency-assurance.md)
- [S4：Hermes 配置与 Context Engine](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- [S5：Hermes Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers/)
- [S6：LanceDB Hermes Memory 插件](https://github.com/lancedb/hermes-agent-memory)
- [S7：Karpathy LLM Wiki 原始 idea file](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [S8：Hermes Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- [S9：hermes-lcm 安装脚本](https://github.com/stephenschoettler/hermes-lcm/blob/8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54/scripts/install.sh)
- [S10：hermes-lcm 配置源码](https://github.com/stephenschoettler/hermes-lcm/blob/8d1b1e6d3d63f5fc7b209e8d7ec1dc9b814f2e54/config.py)
- [Hermes 持久记忆与 session_search](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/)
- [hermes-lcm Releases](https://github.com/stephenschoettler/hermes-lcm/releases)

本文中未标为上游默认值的阈值、实验题数、验收标准、备份周期和 RPO/RTO，均是面向本场景的建议，应通过本地试点校准。
