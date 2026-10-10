# Hermes Agent 进阶使用指南

> 从上下文理解，到能力配置、长期记忆与知识库建设。
>
> 资料核对日期：2026 年 9 月 29 日。
>
> 更新：2026 年 9 月 30 日，补充 server Profile 与 SSH backend 示例，并核对相关官方配置文档。
>
> 同日补充：以 Hindsight 为例，说明 Rerank 模型的训练来源、运行时评分流程及其与记忆更新的区别。
>
> 编辑检查：2026 年 10 月 8 日，全文移除公式，统一图示表达，并检查章节编号、引用与示例。

本指南面向已经能够与 Hermes 正常对话，希望进一步搭建个人助理、研究助手或工作机器人的用户。全文围绕一个案例展开：**搭建一位长期跟踪 AI 行业、记住研究偏好、维护资料库，并通过聊天软件提供服务的研究助手。**

阅读顺序按依赖关系安排：先理解模型如何使用信息，再配置工作身份与能力，最后建立记忆和知识管理体系。

```text
Context：本次任务能读到什么
    ↓
Profile / Bot：按哪套配置工作，从哪里接收请求
    ↓
Skill / MCP：如何完成任务，能够连接哪些系统
    ↓
Hindsight：跨会话保存与找回经验
    ↓
LCM：管理长对话，并按需恢复原始细节
    ↓
LLM Wiki：把资料整理成可维护、可引用的知识
```

本文中的 Hindsight 指可接入 Hermes 的长期记忆系统；LCM 以社区项目 `stephenschoettler/hermes-lcm` 为例；LLM Wiki 指 Hermes 提供的同名 Skill。功能和安装方式会随版本变化，配置时应核对当前版本文档。文中的演示数据不代表实际测试结果或系统默认参数。

## 目录

1. [Context：理解模型每次回答时看到了什么](#1-context理解模型每次回答时看到了什么)
2. [Profile：为不同工作建立独立配置](#2-profile为不同工作建立独立配置)
3. [Bot：把助手接入日常聊天入口](#3-bot把助手接入日常聊天入口)
4. [Skill：将成熟做法整理为可复用流程](#4-skill将成熟做法整理为可复用流程)
5. [MCP：连接外部工具与数据](#5-mcp连接外部工具与数据)
6. [Hindsight：长期记忆与检索模型](#6-hindsight长期记忆与检索模型)
7. [LCM：保留原始记录的上下文管理](#7-lcm保留原始记录的上下文管理)
8. [LLM Wiki：建立可维护的知识库](#8-llm-wiki建立可维护的知识库)
9. [综合实战：持续工作的研究助手](#9-综合实战持续工作的研究助手)
10. [搭建顺序、验收与常见误区](#10-搭建顺序验收与常见误区)
11. [参考资料](#11-参考资料)

## 1. Context：理解模型每次回答时看到了什么

### 1.1 什么是 Context？

Context，通常译为“上下文”，指模型在**当前这次调用中能够读取的信息**。它不只包含用户刚输入的那句话，还可能包括：

- 系统规则与角色要求。
- 当前用户的问题，以及保留在会话中的历史消息。
- 已加载的 Skill 指令与参考资料。
- 工具定义，以及搜索、读取文件等工具返回的内容。
- 从记忆系统或知识库中取回的相关信息。

可以把 Context 想象成一张工作桌：模型根据桌上摆着的材料完成任务。硬盘里存有文件，并不意味着文件内容已经摆上桌。

因此，**聊天记录已保存、记忆库里有事实、Wiki 中有文章，都不代表模型此刻已经读到了它们。**

例如，你上个月说过：

> 我主要关注开源 AI 项目，报告请优先分析部署成本。

今天你问：

> 帮我分析这个新项目。

只有当旧消息仍在当前上下文中，或相关偏好被检索并重新注入，模型才能据此稳定地调整报告。

### 1.2 用人的短期记忆与长期记忆来理解

可以先建立这个直观类比：**Context 相当于人的短期记忆，而向量库和 Wiki 可以充当 Agent 的长期记忆载体。** 更准确地说，Context 接近“工作记忆”：当前正在使用、用来思考和完成任务的信息。

| 人的记忆类比 | Agent 中的对应物 | 特点 |
|---|---|---|
| 短期记忆 / 工作记忆 | Context | 容量有限，直接参与当前推理 |
| 长期记忆中的关联线索 | 向量库支持的记忆检索 | 根据问题的语义找回相关记录 |
| 长期积累的知识体系 | Wiki | 按主题、概念与来源组织可阅读的知识 |
| 回忆并用于当前思考 | 检索后将相关内容放入 Context | 将持久保存的信息转为当前可用的材料 |

例如，你曾告诉助手“我们没有专职运维人员”。这句话可以长期保存在记忆系统中。一个月后，你问“我们适合哪种部署方式”，系统先找回这条约束，再将它放进 Context，模型才会在本次分析中使用它。

```text
长期保存的信息
├── 向量库：按语义关联找回记忆
└── Wiki：按主题和链接读取知识
             ↓ 检索 / 阅读
Context：当前工作记忆
             ↓
本次推理与回答
```

这个类比描述的是功能关系，不表示 Agent 具有与人脑相同的记忆机制。Context 中的信息不会自动变成长期记忆；向量库与 Wiki 中的信息也不会自动出现在每次对话里。系统需要分别实现保存、检索和加载。

此外，向量库和 Wiki 都是模型之外的存储。把一条信息写进去，并不等于重新训练了模型，也不等于改变了模型权重。两者的具体存储差异见第 8.3 节。

### 1.3 上下文窗口与 Token

模型一次能处理的信息量有限，这个容量通常用 Token 衡量。Token 是文本切分单位，不能简单等同于汉字数或英文单词数。

理解上下文预算时，要同时考虑当前输入材料占用的空间，以及为模型回答预留的空间。这两部分都需要在模型与接口允许的容量内安排。

具体的输入、输出及其他计费或容量限制取决于模型与接口。

长对话会不断累积消息、网页正文、文件和工具结果。容量不足时，系统需要选择、裁剪或压缩材料。即使容量足够，过多无关内容也可能让关键约束被噪声淹没。

上下文管理的目标，是让模型在合适的时间读到**相关、可靠且足够的信息**。

### 1.4 如何提供高质量任务上下文？

“研究一下这个项目”只提供了一个模糊目标。更完整的任务可以写成：

> 请分析项目 A 是否适合三人团队部署。我们主要处理中文资料，没有专职运维。请比较部署复杂度、持续成本与数据导出能力，引用官方资料，并列出尚未验证的地方。

这段话提供了四类信息：

| 信息 | 示例 |
|---|---|
| 目标 | 判断是否适合部署 |
| 约束 | 三人团队、中文资料、无专职运维 |
| 比较维度 | 部署复杂度、持续成本、数据导出 |
| 交付标准 | 引用官方来源，标明未验证内容 |

记忆系统可以补充长期背景，但当前任务的新要求仍应明确说明。

## 2. Profile：为不同工作建立独立配置

### 2.1 Profile 管理什么？

Profile 是一套 Agent 工作配置，包括使用什么模型、遵循什么角色要求、加载哪些技能，以及维护哪些本地记忆和会话状态。

| Profile | 职责 | 输出习惯 |
|---|---|---|
| `researcher` | 阅读资料、比较方案 | 强调引用与证据 |
| `writer` | 将研究结果写成文章 | 强调表达与结构 |
| `personal` | 日常安排与个人事务 | 强调简洁和个人偏好 |
| `server` | 通过 SSH 在 `xalnxXXX` 服务器执行任务 | 先确认远端身份、目录和操作范围 |

将不同工作放进不同 Profile，可以减少模型设置、工具连接和角色规则之间的混杂。[Profile 官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/)

### 2.2 建立研究助手

```bash
hermes profile create researcher
hermes -p researcher setup
hermes -p researcher chat
```

这三步分别创建配置、完成初始化，并以该配置开始对话。这里显式使用 `-p researcher`，方便确认命令作用于哪个 Profile。

可以为这个助手约定：

> 优先使用一手资料；区分事实、推断和建议；涉及版本、价格与许可证时注明来源日期；输出结论时解释适用条件。

### 2.3 实战：为 xalnxXXX 建立 server Profile

这个例子建立一个名为 `server` 的 Profile，使用 **SSH terminal backend**，让 Agent 的终端命令在 `xalnxXXX` 服务器上执行。

这里的 `server` 是自定义 Profile 名称，`ssh` 才是 backend 类型。两者的关系是：

```text
Hermes 运行环境
    └── server Profile
          ├── 模型、角色、技能等配置
          └── terminal.backend: ssh
                    ↓ 使用 SSH key 认证
                xalnxXXX server
                    └── 以指定远端用户执行终端命令
```

设置 SSH backend 决定终端命令的执行位置，不等于将整个 Hermes 进程、模型服务和全部工具都迁移到了远端。[Hermes SSH backend 配置](https://hermes-agent.nousresearch.com/docs/user-guide/configuration/#ssh-backend)

#### 第一步：先打通 SSH key 访问，再创建 Profile

**创建 `server` Profile 之前，先确保 Agent 所在的运行环境，能够通过 SSH key 免交互访问 `xalnxXXX` server。** 本例按这个顺序搭建，避免将 SSH 连接问题与 Profile 配置问题混在一起。

这里的“能够访问”不是指你在另一个终端里曾经输入密码登录成功，而是指运行 Hermes 的同一台机器、同一操作系统用户及同一运行环境，能够使用指定密钥完成连接。如果 Hermes 运行在 WSL、容器或后台服务里，测试也应在对应环境完成。

需要准备好：

- `xalnxXXX` 的实际主机名或 IP，以及 SSH 端口。
- 允许登录的远端用户名。
- Hermes 运行环境能够读取的 SSH 私钥文件。
- 服务器端已授权对应公钥，通常配置在该远端用户的 `~/.ssh/authorized_keys` 中。

已有可用密钥时可以复用相应配置；需要新密钥时，可生成专用密钥对，再把 `.pub` 公钥加入服务器授权。私钥保留在发起 SSH 连接的一端，不应写进文档或复制到服务器充当公钥。

下面以 Linux / WSL 中的路径为例。`local-user`、`remote-user`、密钥路径和 `xalnxXXX` 都应替换为实际值；Windows 原生环境使用实际 Windows 路径。

```bash
# 仅在需要新密钥时执行；选择未被占用的文件名，保留已有密钥
ssh-keygen -t ed25519 -f /home/local-user/.ssh/hermes_xalnxXXX -C "hermes-server"
```

通过已有登录方式或服务器管理员，将生成的公钥文件 `hermes_xalnxXXX.pub` 加入目标用户的授权列表。首次连接时核对服务器主机密钥指纹，并完成可信主机记录。公钥认证与主机身份验证是两个不同步骤。[OpenSSH 认证说明](https://man.openbsd.org/ssh)

如果私钥设置了口令，应先通过运行环境可用的 `ssh-agent` 加载密钥，并确保 Hermes 进程能访问同一个 agent；后台服务不一定继承交互式终端中的 agent 环境。

然后，在 Hermes 将使用的环境中执行只读连接测试。下面默认使用 SSH 的 22 端口；非默认端口需增加 `-p 实际端口`。

```bash
ssh -o BatchMode=yes -o PreferredAuthentications=publickey -o IdentitiesOnly=yes -i /home/local-user/.ssh/hermes_xalnxXXX remote-user@xalnxXXX hostname
```

`BatchMode=yes` 禁止交互式认证提示，`PreferredAuthentications=publickey` 将测试限制在公钥认证。命令应直接返回目标服务器的主机名并正常退出，不应等待输入账号密码或密钥口令。[OpenSSH 客户端配置说明](https://man.openbsd.org/ssh_config)

如果连接失败，先检查网络、用户名、公钥授权、密钥路径和密钥代理；通过这一步后再继续创建 Profile。

#### 第二步：创建 server Profile

```bash
hermes profile create server
hermes -p server setup
```

为这个 Profile 配置所需的模型与凭据。可以设置这样的角色约定：

> 你负责 xalnxXXX 服务器上的指定工作。执行任务前确认主机名、登录用户和工作目录；涉及路径时明确区分 Hermes 所在机器与远端服务器；根据用户指定的项目范围操作。

#### 第三步：将 terminal backend 设置为 ssh

```bash
hermes -p server config set terminal.backend ssh
```

默认目录布局下，这对应 `~/.hermes/profiles/server/config.yaml` 中的配置：

```yaml
terminal:
  backend: ssh
```

在同一个 Profile 的 `~/.hermes/profiles/server/.env` 中设置连接信息，保留文件中已有的其他配置：

```dotenv
TERMINAL_SSH_HOST=xalnxXXX
TERMINAL_SSH_USER=remote-user
TERMINAL_SSH_KEY=/home/local-user/.ssh/hermes_xalnxXXX
TERMINAL_SSH_PORT=22
```

| 配置项 | 本例含义 |
|---|---|
| `terminal.backend` | `ssh`，让终端工具通过 SSH 执行命令 |
| `TERMINAL_SSH_HOST` | 目标服务器的实际主机名、IP 或可解析的 SSH 主机别名 |
| `TERMINAL_SSH_USER` | 已通过连接测试的远端用户 |
| `TERMINAL_SSH_KEY` | Hermes 运行环境中的私钥路径，不是远端路径，也不是 `.pub` 文件 |
| `TERMINAL_SSH_PORT` | 服务器的 SSH 端口；默认是 22 |

以上是默认 Profile 目录布局；如果自定义了 Hermes 的主目录，应编辑该 Profile 实际使用的文件。连接配置必须与第一步验证通过的主机、用户、端口和密钥一致。[Hermes Profile 文档](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/)、[SSH 配置参考](https://hermes-agent.nousresearch.com/docs/user-guide/configuration/#ssh-backend)

#### 第四步：验证 Agent 的终端确实在远端执行

```bash
hermes -p server chat
```

在对话中请求：

> 请通过 terminal 工具分别执行 hostname、whoami 和 pwd，报告当前主机、登录用户与工作目录，并确认是否位于 xalnxXXX 服务器。

将返回结果与第一步的 SSH 测试及预期远端用户核对。实际主机名可能与连接别名不同，应确认它们指向同一台服务器。

这里要让 **Agent 调用 terminal 工具**。Hermes CLI 中直接输入 `!hostname` 属于本机 Shell 模式，官方文档说明它不会使用配置的 SSH backend，不能用来验证远端终端。[CLI Shell 模式说明](https://hermes-agent.nousresearch.com/docs/user-guide/cli)

验证通过后，就可以给出实际任务，例如：

> 查看 xalnxXXX 上指定项目目录的结构，确认运行环境，并整理部署说明。

此时，`server` Profile 提供这项工作的配置，SSH backend 提供远端终端执行能力，而 SSH key 是访问服务器的认证基础。

### 2.4 配置隔离的边界

需要区分两个概念：

- **Profile 与用户画像不同。** Profile 是运行配置；用户画像是关于你的偏好与背景的信息。
- **配置隔离不等于完整的安全隔离。** 两个 Profile 如果连接同一个外部记忆 Bank、使用同一套凭据或访问相同目录，仍可能共享数据。

克隆配置后尤其要检查外部连接。官方文档明确指出，克隆的嵌入式 Hindsight 配置可能继续共享原来的守护进程和记忆 Bank，直到为其设置独立的 Hindsight `profile` 和 `bank_id`。[Profile 克隆说明](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/)

## 3. Bot：把助手接入日常聊天入口

### 3.1 Bot 与 Profile 的关系

本文中的 Bot 指通过 Telegram、Discord、Slack 等平台与 Hermes 交互的机器人入口。

```text
你发送消息
    ↓
Bot / 聊天平台
    ↓
Hermes Gateway 接收与路由
    ↓
对应 Profile 下的 Agent 处理任务
    ↓
返回结果
```

**Bot 决定你在哪里使用助手，Profile 决定助手按哪套配置工作。**

### 3.2 接入一个聊天平台

先验证命令行中的对话正常，再配置研究助手的聊天入口：

```bash
hermes -p researcher gateway setup
```

按向导完成平台凭据、允许用户与运行方式等配置。[官方快速入门](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart/)

之后可以在手机上发送：

> 把这篇文章加入研究资料，告诉我它与我们之前关注的项目有什么联系。

Bot 接收请求，后面的 Agent 和工具负责阅读、检索记忆与维护知识库。

### 3.3 使用时注意什么？

Hermes Gateway 提供白名单和配对授权机制。应明确哪些用户可以使用 Bot，以及它连接了哪些工作资源。[Gateway 授权说明](https://hermes-agent.nousresearch.com/docs/user-guide/security)

另外：

- Bot 入口本身不会自动创造长期记忆，仍需配置相应记忆机制。
- 能接收消息不代表会定时执行任务；每周推送报告还需要单独设置调度和投递目标。
- 多个入口是否共享会话或记忆，取决于路由、Profile 与记忆 Bank 的实际配置。

## 4. Skill：将成熟做法整理为可复用流程

### 4.1 Skill 是工作方法的载体

Skill 是 Agent 按需加载的工作说明，可以包含适用场景、执行步骤、参考资料、输出格式，也可以配套脚本。

如果每次评估 AI 项目都要重新解释方法，可以把稳定做法整理成 `project-review` Skill。下面是内容示例，不是完整安装包：

```text
适用场景：
评估一个 AI 开源项目。

工作流程：
1. 阅读官方说明、许可证和部署文档。
2. 确认项目解决的问题与目标用户。
3. 比较部署复杂度、成本和可维护性。
4. 区分官方声明、实测结果与推测。
5. 输出结论、证据链接与待验证问题。
```

以后可以请求：

> 使用 project-review Skill，评估项目 A。

### 4.2 渐进加载如何节省上下文？

Hermes 的 Skill 采用渐进加载：先提供简要目录，任务需要时再读取完整说明和相关文件。完整技能内容不必始终占用上下文，但目录和已加载内容仍有上下文成本。[Skills 系统](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills/)

```text
技能目录：有哪些技能、分别适用于什么
    ↓ 按需
SKILL.md：具体工作步骤与要求
    ↓ 按需
参考文件、示例或脚本
```

### 4.3 哪些内容适合写进 Skill？

Skill 适合保存“怎么做”，具体项目事实更适合放进知识库。

| 内容 | 建议位置 |
|---|---|
| 报告必须检查许可证 | Skill |
| 项目 A 当前使用的许可证 | 有来源和日期的 Wiki 页面 |
| 用户偏好简洁的中文报告 | 长期偏好记忆或角色约定 |
| 本次报告明天交付 | 当前任务上下文 |

这样，事实变化时不必反复修改工作方法。

## 5. MCP：连接外部工具与数据

### 5.1 MCP 提供什么？

MCP，全称 Model Context Protocol，是模型应用连接外部工具与数据的一种标准协议。

通过 MCP，Hermes 可以连接文件系统、数据库、代码托管平台或内部业务系统。Hermes 支持本地和远程 MCP 服务，并可限制暴露给 Agent 的工具。[MCP 官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)

常见连接方式包括：

- **本地 stdio 服务：** Hermes 启动本地进程，通过标准输入输出通信。
- **远程 HTTP 服务：** Hermes 连接一个服务地址，并按服务要求完成认证。

MCP 是连接协议，不会自动替你创建外部账户、授予权限或确保返回内容正确。

### 5.2 Skill 与 MCP 如何配合？

| 组件 | 主要作用 | 研究助手示例 |
|---|---|---|
| Skill | 指导工作流程 | 说明如何评估开源项目 |
| MCP | 提供系统连接与工具 | 获取项目 Issue、读取资料 |
| Context | 承载本轮实际读取的内容 | 工具取回的 Issue 摘要 |

例如：

> 按项目评估流程，检查项目 A 最近的维护情况。

Skill 指导助手查看哪些证据，MCP 可以提供访问对应系统的工具，工具结果进入 Context 后，模型才能进行分析。

### 5.3 从最小需要开始连接

优先接入当前任务真正需要的服务，限制目录范围、访问权限和工具数量。额外工具会增加选择复杂度与维护成本。已有内置工具能完成的任务，也不必额外添加 MCP。[MCP 使用指南](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes/)

## 6. Hindsight：长期记忆与检索模型

### 6.1 Hindsight 解决什么问题？

当前对话能继续，不等于新会话也知道过去的事情。Hindsight 帮助 Agent 保存和检索跨会话记忆。

| 操作 | 含义 | 示例 |
|---|---|---|
| Retain | 保存并提取记忆 | 记录用户主要研究开源 AI |
| Recall | 查找相关记忆 | 找回团队的部署约束 |
| Reflect | 综合已有记忆进行分析 | 总结过去几次选型的共同原因 |

Hermes 当前文档提供的接入方式是：

```bash
hermes plugins install hindsight
hermes memory setup
```

在向导中选择 Hindsight，再配置云服务、嵌入式本地服务或已有服务。若用于前面的研究 Profile，可在命令中显式加入 `-p researcher`。

Hindsight 提供上下文自动注入、工具调用及混合等使用模式。自动保存、自动召回是否开启，需要检查实际配置，不能仅凭安装完成就假定已经生效。[Hermes 记忆提供方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)

### 6.2 区分三种模型的工作

长期记忆系统中，不同模型承担不同任务：

| 模型 | 主要任务 | 输出 |
|---|---|---|
| Embedding Model（向量模型） | 将文本编码为适合检索的数值表示 | 向量 |
| Rerank Model（重排序模型） | 判断候选内容对当前问题的相关程度 | 分数或排序 |
| 生成式 LLM | 提取事实、综合材料、组织回答 | 文本或结构化结果 |

它们可以分别配置。更换聊天模型，并不意味着记忆检索模型也随之改变。

### 6.3 Embedding Model：把文本映射到可检索的向量空间

#### 从文本到向量

以常见的稠密文本向量模型为例，它会把一段文本转换成固定维度的向量：

```text
“我们没有专职运维人员”
            ↓
    Embedding Model
            ↓
[0.12, -0.38, 0.07, …]
```

这些数字是模型学习到的表示。通常不能把某一维简单解释为“成本”，另一维解释为“运维”。

常见的处理过程是：

```text
文本 → 分词 → Transformer 编码 → 聚合为文本向量 → 可选归一化
```

具体聚合方式因模型而异，例如对 Token 表示求平均，或使用特定位置的表示。

在检索中，问题和候选内容通常分别编码：

```text
问题 ──→ 查询编码 ──→ 问题向量
记忆 ──→ 内容编码 ──→ 记忆向量
                         ↓
                 计算相似度并检索
```

这种方式通常称为 **Bi-Encoder，双编码器**。“双”指问题和内容分别处理，不一定是两套不同权重的模型；同一模型也可能针对问题与文档使用不同的前缀或编码设置。[Sentence Transformers 架构说明](https://www.sbert.net/examples/cross_encoder/applications/README.html)

#### Hindsight 例子：保存团队约束，再按语义找回

假设你告诉 Hermes：

> 项目 P 由三个人负责，主要处理中文资料。我们没有专职运维人员，试点部署要尽量简单。

当这段内容被提交给 Hindsight 的 `retain` 操作时，系统会保留来源内容，并提取结构化事实、实体和时间等信息，然后为相关事实生成向量、建立关联并保存。下面是可能提取出的事实示例，不是实际运行结果；具体提取内容取决于模型与配置。[Hindsight Retain 说明](https://hindsight.vectorize.io/developer/retain)

| 示例事实 | Embedding 的作用 |
|---|---|
| 项目 P 由三个人负责 | 将团队规模信息编码为可检索的向量 |
| 项目 P 没有专职运维人员 | 将运维资源约束编码为可检索的向量 |
| 项目 P 的试点部署需要尽量简单 | 将部署要求编码为可检索的向量 |

```text
用户提供的内容
    ↓ Retain
LLM 提取事实、识别实体等
    ↓
Embedding Model 为事实生成向量
    ↓
保存事实文本、向量与来源等信息
```

一个月后，你开启新会话，问：

> 项目 P 适合哪种部署方式？我们希望减少维护工作。

Hindsight 的语义检索会将当前查询也编码为向量，再查找记忆库中的相关记录。“减少维护工作”与“没有专职运维人员”措辞不同，但语义有关，因此后者有机会成为召回候选。这不要求问题与记忆包含完全相同的关键词。

```text
当前查询：“项目 P 如何减少维护工作？”
    ↓
Embedding Model 生成查询向量
    ↓
语义检索召回相关记忆
    ↓
与其他已启用检索方式的结果融合，再按配置重排序
    ↓
返回记忆文本，由 Hermes 按记忆模式使用
```

Hindsight 的 `recall` 采用四路检索架构：Semantic 向量检索、Keyword 的 **BM25 关键词检索**、Graph 关系图检索，以及 Temporal 时间范围检索。Embedding 负责其中的语义检索部分；它本身不替用户决定部署方案。被找回的团队约束进入 Context 后，生成式 LLM 才会结合当前项目资料分析方案。四路检索的分工见第 6.6 节。[Hindsight Recall 示例](https://hindsight.vectorize.io/cookbook/recipes/quickstart)

**中文场景中的模型配置示例：**

Hindsight 的多语言文档给出了使用 `BAAI/bge-m3` 作为本地 Embedding 模型的配置方式：

```dotenv
HINDSIGHT_API_EMBEDDINGS_PROVIDER=local
HINDSIGHT_API_EMBEDDINGS_LOCAL_MODEL=BAAI/bge-m3
```

这是一个可选择的多语言配置，不表示你的实例已经安装或启用了它。配置应在 Hindsight 服务实际运行的环境中生效。重排序模型的中文能力也需要配套评估。[Hindsight 多语言配置](https://hindsight.vectorize.io/developer/multilingual)

在这个过程中，Embedding 模型是在用已训练的权重生成向量。新增一条记忆增加的是存储内容及其向量，不会自动用这段聊天重新训练 Embedding 模型。

#### 模型如何学会语义检索？

常见训练方式是对比学习：给模型提供问题、相关内容，以及不相关或容易混淆的内容，让相关配对的得分更高。具体训练目标与数据因模型而异。

| 训练角色 | 示例 |
|---|---|
| 问题 | 小团队适合什么部署方式？ |
| 正样本 | 无专职运维的团队可优先评估托管方案 |
| 困难负样本 | 大型团队维护自建集群的实践 |

困难负样本同样谈部署，但不直接匹配问题。学习这种区分，比只识别几个共同关键词更有价值。

#### 为什么能快速检索大量记忆？

记忆入库时，可以提前计算并保存向量。查询时，只需计算问题向量，再通过向量索引寻找相近内容，无须对每条记忆重新运行一次文本编码。

大规模系统通常使用近似最近邻索引降低检索开销。这也意味着检索结果同时受模型、索引参数与检索范围影响，不能将所有漏召回都归因于模型。

向量需要存储空间。例如，保存 100 万条 1,024 维向量，并以 32 位浮点数保存每个数值时，仅原始向量就需要约 4.1 GB。这还未包含原文、索引和数据库开销。

**维度不是质量分数。** 更高维不保证中文检索更准，语言覆盖、训练数据与任务适配同样关键。以 BGE-M3 为例，其稠密表示为 1,024 维，并同时提供稀疏与多向量检索能力，说明 Embedding 模型并不只有一种输出机制。[BGE-M3 模型说明](https://huggingface.co/BAAI/bge-m3)

#### 为什么仍然会找错？

向量将文本映射成便于比较的表示，但语义接近不等于答案相同。例如：

- “团队曾计划采用方案 B。”
- “团队最终放弃了方案 B。”

两句话主题接近，决策状态却不同。单靠相似度可能无法稳定处理所有否定、时间与条件差异，这也是引入重排序的原因。

### 6.4 Rerank Model：联合判断问题与候选内容

#### Cross-Encoder 的结构

向量检索适合快速缩小范围。接下来，Rerank Model 对已经找到的候选进行更细致的相关性判断。

一种常见实现是 **Cross-Encoder，交叉编码器**：将问题与一条候选内容一起输入模型，通过同一次 Transformer 编码得到相关性分数。

```text
当前问题与一条候选记忆
        ↓
同一次 Transformer 编码
        ↓
相关性评分层
        ↓
该候选对当前问题的分数
```

模型每处理一组问题与候选记忆，就返回该候选对当前问题的相关性分数。处理全部候选后，系统便可以依据分数重新排序。

问题和候选的 Token 在同一次计算中交互。相比各自编码后比较向量，这种方式有机会更细致地判断因果、否定、主体与条件是否匹配，但不保证判断总是正确。[Cross-Encoder 使用说明](https://sbert.net/docs/cross_encoder/usage/usage.html)

例如，问题是：

> 为什么最终没有采用方案 B？

候选内容包括：

| 候选 | 内容 | 与问题的关系 |
|---|---|---|
| A | 方案 B 的部署费用较低 | 背景信息 |
| B | 团队曾计划采用方案 B | 中间状态 |
| C | 最终因无法完整导出数据而放弃方案 B | 直接解释最终决定 |

重排序的目标是让 C 排到更前面。这里展示的是理想排序，并非实测结果。

#### Rerank Model 如何训练？以 Hindsight 的实际用法说明

先区分两个阶段：**模型训练负责让模型学会判断相关性；Hindsight 的常规使用流程负责调用已经训练好的模型，为当前候选记忆评分。** Hindsight 不会因为用户多聊了几次，就自动用这些聊天更新 Rerank 模型的权重。

**一、训练发生在哪里？**

截至本次核对，Hindsight 官方配置列出的本地默认 provider 是 `local`，默认模型字符串为 `cross-encoder/ms-marco-MiniLM-L-6-v2`。对应的 MiniLM 重排序模型由模型提供方预先训练，模型卡说明其训练任务是 **MS MARCO Passage Ranking**，即根据问题判断哪些文本段落相关。[Hindsight 配置说明](https://github.com/vectorize-io/hindsight/blob/main/skills/hindsight-docs/references/developer/configuration.md)、[MiniLM 模型卡](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2)

为了理解训练信号，仍以问题“为什么最终没有采用方案 B？”为例。假设我们制作了下面的训练数据：

| 问题对应的候选段落 | 示例标签 | 标注理由 |
|---|---|---|
| 最终因无法完整导出数据而放弃方案 B | 相关 | 直接回答放弃原因 |
| 方案 B 的界面比较美观 | 不相关 | 没有解释最终决定 |
| 团队曾计划采用方案 B | 不相关 | 只说明早期计划，没有回答原因 |

这张表是教学用的合成样例，不是 Hindsight 的训练记录，也不是上述模型实际使用的 MS MARCO 样本。“不相关”是相对于这个具体问题而言，不代表内容本身没有价值。

**二、Hindsight 实际怎样调用模型？**

本地实现通过 Sentence Transformers 的 `CrossEncoder` 加载模型，并对“问题—候选文本”配对进行预测；远程 provider 则调用相应服务。这里执行的是推理评分。[Hindsight 重排序实现](https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/cross_encoder.py)

假设记忆库中已经保存了以下三条记录，用户现在问“为什么最终没有采用方案 B？”：

| 记忆 | 时间 | 内容 |
|---|---|---|
| M1 | 9 月 1 日 | 团队计划评估方案 B，主要关注部署成本 |
| M2 | 9 月 10 日 | 因方案 B 无法完整导出数据，团队决定放弃 |
| M3 | 9 月 20 日 | 方案 B 发布了界面改进，尚未重新评估 |

一次简化的 Hindsight 处理过程如下：

1. **召回候选。** 根据已启用的 Semantic 向量检索、Keyword（BM25）、Graph 和 Temporal 检索找到候选，并融合排名。假设 M1、M2、M3 都进入候选集。
2. **准备评分文本。** 将每条记忆的正文作为候选文本；有上下文标签或起始日期时，可将这些信息加到正文前面。
3. **联合评分。** 将当前问题分别与 M1、M2、M3 配对处理，为每条候选记忆得到相关性分数。
4. **整理分数。** 按 provider 的输出类型使用已归一化分数，或对原始 logit 使用 Sigmoid 转换；结合适用的时间与证据等信号调整排名。
5. **返回相关记忆。** 按请求的筛选条件和返回预算交给调用方；Hermes 再根据记忆模式使用这些结果。

上述流程依据 Hindsight 官方重排序说明；具体细节应以运行版本为准。[Hindsight 重排序技术说明](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory)

以下数值仅演示重排效果，**不是实测结果，也不是最终综合分数**：

| 记忆 | 假设的召回次序 | 示例相关性分数 | 示例重排次序 |
|---|---|---|---|
| M1：早期评估计划 | 1 | 0.28 | 2 |
| M2：因无法导出数据而放弃 | 2 | 0.93 | 1 |
| M3：近期界面更新 | 3 | 0.12 | 3 |

这个例子中，M2 直接回答了“为什么放弃”。M3 虽然更新，却没有解释过去的决定。实际结果由所用模型、数据、检索范围和配置共同决定，不能保证任何模型都输出这个顺序。

**三、在 Hindsight 中配置的是推理模型**

以下示例展示官方文档中的本地重排序配置，而不是训练命令：

```dotenv
HINDSIGHT_API_RERANKER_PROVIDER=local
HINDSIGHT_API_RERANKER_LOCAL_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
HINDSIGHT_API_RERANKER_MAX_CANDIDATES=300
```

`MAX_CANDIDATES` 限制一次重排序处理的候选数量，不是训练样本数，也不是最终注入 Context 的记忆条数。上述数值来自当前官方配置示例，不代表你的实例一定使用这些值。[Hindsight 环境配置示例](https://github.com/vectorize-io/hindsight/blob/main/.env.example)

这些设置需要在 **Hindsight 服务实际运行的环境**中生效。云端服务、本地嵌入式服务和独立部署的配置入口可能不同，不能假定改 Hermes 的聊天模型配置就能切换重排序模型。

默认示例中的 MiniLM 模型以英文段落排序任务训练；本节以中文解释流程，并不构成该默认模型的中文效果证明。[模型语言与训练任务](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2)

#### 为什么不直接检查整个记忆库？

Cross-Encoder 的评分依赖当前问题，无法像记忆向量一样提前为所有未来问题算好。

如果有 100 万条记忆，对每条都与问题联合编码，计算量会很大。因此通常先检索少量候选，再进行重排序。多个候选可以批量推理，但总计算量仍随候选数量和文本长度增加。[检索与重排序流程](https://www.sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)

**相关性分数不是事实正确率。** 过时甚至错误的记忆也可能非常贴合问题。即使分数归一化到 0～1，也不能直接解释为“有 95% 的概率是真实的”。

### 6.5 Embedding 与 Rerank 的对比

| 维度 | Embedding Model | 典型 Cross-Encoder Rerank Model |
|---|---|---|
| 处理方式 | 分别编码问题和内容 | 联合编码问题与候选 |
| 主要输出 | 可索引的向量 | 相关性分数 |
| 文档侧能否预计算 | 可以 | 无法为所有未来问题预计算评分 |
| 适用阶段 | 大范围召回 | 候选集合内精细排序 |
| 主要优势 | 适合大规模检索 | 能进行更细的配对判断 |
| 主要限制 | 可能遗漏细节或混淆相近含义 | 成本随候选数量与长度增加 |
| 更换模型的影响 | 通常需要重新生成记忆向量 | 通常不需要重建记忆向量 |

### 6.6 Hindsight 的四路检索如何与 Embedding、Rerank 协作？

**Hindsight 的 Recall 明确使用 BM25，但整体召回同时结合四种检索策略。** 官方快速入门将它描述为并行运行的四路检索；实际执行哪些策略，还取决于服务配置、Bank 设置与当前查询。[Hindsight 四路检索说明](https://hindsight.vectorize.io/cookbook/recipes/quickstart)

| 检索策略 | 使用的信息 | 作用 |
|---|---|---|
| **Semantic：语义检索** | Embedding 向量及其相似性 | 找到措辞不同、含义相关的记忆 |
| **Keyword：关键词检索** | **BM25 关键词匹配与排序** | 找到包含项目名、术语或其他查询词的记忆 |
| **Graph：关系图检索** | 实体、时间与因果等关系链接 | 沿已建立的关系找到关联记忆 |
| **Temporal：时间范围检索** | 记忆的时间信息及查询中的时间范围 | 找到对应时段内的记忆 |

BM25 是关键词相关性排序方法，无需先生成 Embedding。它与 Semantic 互补：关键词检索利用文字匹配，语义检索帮助覆盖同义表达。Graph 与 Temporal 则进一步利用关系和时间信息。[Hindsight Recall 策略说明](https://github.com/vectorize-io/hindsight/blob/main/skills/hindsight-docs/references/developer/api/recall.md)

例如，你问：

> 9 月项目 P 为什么放弃了方案 B？

在相关记忆已经保存、对应策略已启用的前提下，四路检索可以分别提供这样的候选线索：

| 策略 | 本例中的线索 |
|---|---|
| Semantic | 找到“由于数据迁移受限，团队改选其他方案”等语义相关内容 |
| Keyword（BM25） | 找到包含“项目 P”“方案 B”等名称的记录 |
| Graph | 沿项目、方案与决策原因之间已保存的关联查找记忆 |
| Temporal | 根据明确的年份及 9 月范围查找当时的记录 |

这些是功能示例，不代表实际命中结果。同一条记忆可以被多路召回；某一路没有找到，也可能由其他路找到。Graph 中的时间关联用于沿关系查找，Temporal 则侧重时间范围，两者的侧重点不同。

整个简化流程如下：

```text
记忆写入：
原始材料 → 事实提取与组织 → 记忆文本 → Embedding → 保存与索引

记忆查询：
当前问题
   ├── Semantic：Embedding 向量相似度检索
   ├── Keyword：BM25 关键词检索
   ├── Graph：实体 / 时间 / 因果关系图检索
   └── Temporal：时间范围检索
   ↓
候选合并与排名融合（RRF）
   ↓
Rerank Model 评分与重排
   ↓
按预算选择相关记忆
   ↓
交给生成式 LLM 使用
```

RRF（Reciprocal Rank Fusion，倒数排名融合）根据各路结果的名次融合候选排名。之后，Rerank Model 再联合读取问题和候选文本，进行更细的相关性评分；还可结合时间和证据等信号调整排名。具体机制取决于版本与配置。[Hindsight 重排序技术说明](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory)

以下数字仅用于说明流程，不是默认配置：

```text
10 万条记忆 → 检索出 100 条候选 → 重排序 → 选择 8 条 → 进入上下文
```

关键边界是：**重排序只能处理进入候选集合的内容。相关记忆如果在召回阶段被漏掉，Rerank Model 无法凭空找回。**

### 6.7 长期记忆的维护原则

Hindsight 的记忆提取与综合分析可能使用独立 LLM。本地部署仍有计算与存储成本；如果调用外部模型，还会产生相应调用费用。[Hindsight 部署说明](https://hindsight.vectorize.io/developer/installation)

记忆质量也取决于保存策略：

- 保存长期有效的偏好、约束、决定及其理由。
- 区分用户已确认的事实与助手自己的推测。
- 记录决定的时间和适用范围。
- 偏好或方案发生变化时，明确标记新旧关系。

例如，“本周试点暂用方案 A”不应被保存成“用户始终偏好方案 A”。

## 7. LCM：保留原始记录的上下文管理

### 7.1 LCM 的“无损”是什么意思？

LCM 的完整名称是 **Lossless Context Management，无损上下文管理**。

长对话需要压缩历史内容。普通摘要可能留下“讨论过三种部署方案”，却省略某个关键参数或排除理由。

LCM 的核心思路是：

1. 保存原始消息。
2. 为较早的消息生成摘要。
3. 建立摘要与来源之间的关联。
4. 在活跃上下文中保留摘要和近期内容。
5. 需要细节时，搜索并展开原始记录。

`hermes-lcm` 使用 SQLite 保存记录，并通过摘要 DAG 组织历史。DAG 是有向无环图，在这里可以理解为一种带来源关联、能够逐层展开的摘要结构。[hermes-lcm 项目](https://github.com/stephenschoettler/hermes-lcm)

```text
活跃上下文：较高层摘要与近期消息
                 │
                 ↓ 按需展开
            较细的摘要节点
                 │
                 ↓ 追溯来源
              原始消息
```

**摘要仍可能省略信息。“无损”强调原始内容被保留，并具有恢复路径；它不保证模型始终记得所有细节，也不保证每次都会主动找回。**

### 7.2 举例：找回被摘要省略的约束

假设最初确定：

> 试点阶段最多接入 20 个数据源；只有完成质量评估后才能扩大范围。

长对话压缩后，摘要可能只剩：

> 已讨论数据接入计划。

如果 Agent 需要确定规模限制，就应搜索并展开原始消息，而不是从简短摘要猜测。该实现提供 `lcm_grep`、`lcm_describe`、`lcm_expand` 等工具辅助恢复细节。[恢复机制说明](https://github.com/stephenschoettler/hermes-lcm/blob/main/README.md)

### 7.3 配置形态

安装对应插件后，基本配置形态如下：

```yaml
plugins:
  enabled:
    - hermes-lcm

context:
  engine: lcm
```

应将其合并进目标 Profile 的现有配置，并保留其他已启用插件。这个片段不包含安装步骤；具体安装方法与兼容要求以所选实现的项目说明为准。其他 LCM 实现可能使用不同插件名、工具或配置项。

### 7.4 LCM 与 Hindsight 的分工

| 需求 | 优先使用 |
|---|---|
| 找回某次对话中的原话、参数与细节 | LCM |
| 新会话中找回长期偏好或过去经验 | Hindsight |
| 长对话持续推进，减少摘要后的信息断层 | LCM |
| 从多次经历中综合共同规律 | Hindsight 的记忆综合能力 |

这是用途侧重点，不是绝对能力边界。LCM 的某些实现也支持历史会话检索，但它的主要设计目标仍是上下文管理与来源恢复。

两者可以同时使用，也会增加存储、摘要和检索开销。是否引入 LCM，应看长对话中是否确实存在细节恢复需求。

## 8. LLM Wiki：建立可维护的知识库

### 8.1 从保存资料到组织知识

LLM Wiki 让 Agent 将资料整理为互相关联的知识页面，并随着新资料进入持续维护这些页面。

它关注的问题是：

> 关于这个主题，我们目前知道什么？依据是什么？有哪些矛盾或未解决的问题？

研究助手持续阅读 AI 部署资料，可以逐渐形成以下内容。这里是示意结构，实际目录可遵循所用 Skill 的约定：

```text
研究知识库/
├── 原始资料与来源/
├── 项目介绍/
├── 技术概念/
├── 方案比较/
└── 待验证问题/
```

Hermes 的 `llm-wiki` Skill 使用相互链接的 Markdown 文件维护知识库，可用普通编辑器或 Obsidian 打开，并支持通过 `WIKI_PATH` 指定位置。[LLM Wiki 官方说明](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/research/research-llm-wiki)

### 8.2 如何让助手更新 Wiki？

可以直接提出：

> 使用 llm-wiki Skill，把这三份官方资料加入研究知识库。保留来源和日期，更新已有项目页面；如果与旧结论冲突，请记录冲突，不要直接覆盖。最后列出修改过的页面。

一个实用的项目页面，应能区分：

- 来源明确的事实。
- 根据资料做出的推断。
- 已失效的旧信息。
- 尚未验证的问题。

例如，“支持数据导出”还应说明对应版本、导出范围与来源，避免后来被扩展理解为“所有数据都可完整迁移”。

### 8.3 向量库存储与 Wiki 存储有什么区别？

**向量库主要解决“如何按语义找到信息”，Wiki 主要解决“如何把知识组织成可阅读、可维护的页面”。** 它们处于不同层次：前者是一类存储与检索基础设施，后者是一种知识组织方式，可以组合使用。

在典型的文本向量检索系统中，一条记录通常关联三类内容：

1. **文本或文本引用：** 一段原文、提取的事实，或指向原文的标识。
2. **Embedding 向量：** 用于相似度计算与检索。
3. **元数据：** 来源、时间、项目、标签、访问范围等。

这些内容可以存放在同一数据库中，也可以分开放置并通过 ID 关联。**向量库并不意味着只存一串数字。向量也不是原文的无损压缩，不能依靠它还原原文。** 例如，pgvector 允许在 PostgreSQL 中把向量与其他数据一起保存，并执行相似度查询。[pgvector 官方说明](https://github.com/pgvector/pgvector)

本文的 LLM Wiki 则以 Markdown 页面作为主要知识载体，用标题、段落、链接与来源组织信息；向量索引不是它的必需部分。[LLM Wiki 官方说明](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/research/research-llm-wiki)

| 对比维度 | 典型文本向量库方案 | 本文的 Markdown Wiki 方案 |
|---|---|---|
| 主要存储单元 | 文本片段或事实记录，以及对应向量和元数据 | 主题页面、概念页面、比较页面等文件 |
| 组织依据 | 记录 ID、元数据，以及向量表示的相似性 | 目录、标题、主题与显式页面链接 |
| 信息粒度 | 常按事实或适当长度的片段保存 | 常将同一主题的内容组织成连贯页面 |
| 查找方式 | 将问题向量化，结合过滤条件检索；可接重排序 | 按目录、链接、关键词查找，也可额外建立向量索引 |
| 人类阅读 | 文本和元数据可读，向量本身不便直接理解 | 页面内容可以直接阅读和编辑 |
| 写入工作 | 提取或分块、生成向量、保存记录 | 归类、整理、交叉引用、更新页面 |
| 内容更新 | 文本改变后通常需更新对应向量与元数据 | 修改页面，并检查相关链接与结论是否仍成立 |
| 来源追溯 | 依赖记录保存的来源 ID、地址或关联关系 | 依赖页面中保留的引用和原始资料链接 |
| 备份与迁移 | 保存文本、元数据和向量配置；索引可按需重建 | Markdown 与资料文件可直接复制，也可用版本管理 |
| 适合的问题 | “我以前提过哪些类似要求？” | “我们对这个主题已经形成了什么认识？” |

这些是常见设计取向，不是硬性限制：向量库可以保存整篇文档，Wiki 也可以只写一条简短事实。是否保留来源、是否处理冲突，取决于维护流程，任何一种存储都不会自动保证内容正确。

### 8.4 同一条信息，两种方式怎样保存？

假设用户说：

> 项目 P 的试点阶段先采用方案 A，因为我们没有专职运维；如果规模扩大，再重新比较 A 和 B。

**向量库方案：保存可检索的记忆记录。**

下面是概念示例，不是 Hindsight 的实际数据库结构或可直接导入的配置。`embedding_preview` 仅展示完整向量的前三个维度，数值为演示占位值；完整维度以所用模型为准，例如 BGE-M3 的稠密向量为 1,024 维。

```yaml
id: memory-001
text: 项目 P 试点采用方案 A，原因是缺少专职运维；规模扩大后重新评估 A 与 B。
# 仅展示前三个维度，数值为演示占位值
embedding_preview: [0.12, -0.38, 0.07]
metadata:
  project: P
  stage: 试点
  source: 用户确认的会话记录
  status: 当前决定
```

查询“项目 P 为什么在试点阶段采用方案 A”时，可以先限定项目 P，再通过语义检索找到这条相关记忆。得到的通常是一组记录，后续由 Agent 结合来源和其他材料组织答案。

**Wiki 方案：维护项目 P 的部署决策页面。**

该页面可以呈现为：

| 页面栏目 | 内容示例 |
|---|---|
| 当前决定 | 试点阶段采用方案 A |
| 决策依据 | 团队没有专职运维人员 |
| 适用范围 | 当前试点阶段，不代表长期固定选择 |
| 重新评估条件 | 规模扩大后重新比较 A 与 B |
| 相关页面 | 方案 A 介绍、方案 B 介绍、部署方式比较 |
| 来源 | 用户确认记录及其日期 |

查询“项目 P 的部署策略是什么”时，助手可以读取这个页面，直接获得经过组织的整体说明。

如果用户后来改为方案 B，两种方式都需要维护：向量库中应区分新旧决定、更新有效状态；Wiki 中应更新当前方案，并在需要时保留历史理由。不能让旧信息与新信息以同样的“当前事实”身份并存。

### 8.5 三种信息系统的定位

| 系统 | 主要回答的问题 | 典型内容 |
|---|---|---|
| Hindsight | 我们过去有哪些偏好、经历和决定？ | 团队部署约束、选型经验 |
| LCM | 那次对话究竟说了什么？ | 原话、参数、讨论过程 |
| LLM Wiki | 关于这个主题，目前有哪些可引用的知识？ | 项目介绍、概念解释、方案比较 |

在三种系统中保存不同层次的信息，比把同一段完整对话无差别复制三遍更容易维护。

## 9. 综合实战：持续工作的研究助手

### 9.1 一次完整任务

假设你通过 Bot 发来：

> 比较项目 A 与项目 B 是否适合我们部署，并更新研究知识库。

建议的工作过程如下：

| 步骤 | 使用的组件 | 作用 |
|---|---|---|
| 1 | Bot、Gateway | 接收消息，交给研究 Profile |
| 2 | Profile | 确定模型、规则与工具配置 |
| 3 | Hindsight | 找回三人团队、中文资料、运维资源有限等约束 |
| 4 | Skill | 加载项目评估流程 |
| 5 | MCP 或内置工具 | 获取最新官方资料与项目状态 |
| 6 | Context | 汇集本次判断实际需要的信息 |
| 7 | LCM | 长任务中管理历史，必要时恢复早期细节 |
| 8 | LLM Wiki | 更新项目和比较页面，保留来源 |
| 9 | Hindsight | 按记忆策略保存最终决定及其适用条件 |

这是一种工作设计，不是安装所有组件后必然出现的执行顺序。是否自动检索、写入或加载，取决于实际配置、技能流程和任务需要。未做出的决定也不应提前写成已确认记忆。

### 9.2 同一个决定怎样保存？

假设你最后确认：

> 试点阶段先采用 A，因为它部署简单。等数据量超过当前规模后，再评估 B。

三种系统分别保存不同表达：

| 位置 | 保存内容 |
|---|---|
| LCM 原始记录 | 完整讨论与用户确认原话 |
| Hindsight | 当前试点选择 A，原因是部署简单；扩大规模后需重新评估 |
| Wiki | A、B 的技术比较、来源、适用条件和待验证项 |

这样，未来问“我当时怎么说的”“我们现在用什么”“A 和 B 有什么区别”，就能分别走向适合的信息来源。

### 9.3 可复用的任务提示示例

> 请使用研究 Profile 的项目评估流程，比较 A 与 B。先检查与本项目相关的长期约束，再读取 Wiki 中已有的比较内容，并用最新官方资料补充。若历史记录与当前要求冲突，以我本次明确提出的要求为准，并说明差异。输出结论、证据和未验证项；更新 Wiki 时保留来源，不把你的建议写成我已经确认的决定。

## 10. 搭建顺序、验收与常见误区

### 10.1 按实际问题逐层搭建

| 阶段 | 完成的事情 | 简单验收 |
|---|---|---|
| 1 | 建立 Profile，明确任务上下文 | 能稳定输出符合要求的报告 |
| 2 | 配置 Bot | 能从聊天平台完成同一任务 |
| 3 | 固化 Skill，连接必要工具 | 相似任务按一致流程完成 |
| 4 | 接入 Hindsight | 新会话找回指定偏好，偏好更新后能正确使用 |
| 5 | 长对话有需要时引入 LCM | 压缩后仍能找回早期原话或参数 |
| 6 | 资料积累后建立 Wiki | 回答可追溯来源，新资料能更新旧结论 |

### 10.2 三道验收题

**检查 Hindsight：**

> 我对部署方案有哪些长期要求？请区分当前有效要求与过去的临时条件。

检查是否找回正确、当前有效的约束，而非只找到主题相近的旧信息。

**检查 LCM：**

> 刚才讨论中，试点数据源数量上限是多少？请核对原话。

检查压缩后是否真正恢复了来源，而非根据摘要猜测。

**检查 Wiki：**

> 我们目前关于项目 A 的结论是什么？每项结论依据哪里？

检查页面是否保留来源、时间与适用条件。

### 10.3 常见误区

| 误区 | 更准确的理解 |
|---|---|
| 聊天记录存下来了，模型就一定知道 | 还需要让相关内容进入当前上下文 |
| 换一个更强的聊天模型就能解决记忆问题 | 还要检查保存、召回、重排序与注入链路 |
| Embedding 维度越高越好 | 应结合语言、任务、延迟与存储成本验证 |
| Rerank 可以弥补所有漏召回 | 它只能处理已经进入候选集合的内容 |
| Rerank 分数高说明内容真实 | 高分主要表示与问题相关，仍需核对事实与时间 |
| LCM 的摘要不会丢信息 | 摘要会省略，原始记录与恢复路径负责保留细节 |
| Profile 自动隔离所有数据 | 外部服务、凭据和共享目录仍可能共享 |
| Skill 会自动获得操作系统的权限 | Skill 指导流程，实际操作仍依赖可用工具与权限 |
| MCP 接得越多越强 | 应选择任务真正需要的工具与访问范围 |
| Wiki 写入一次就永久正确 | 需要持续维护来源、冲突、版本与失效信息 |

进阶配置最终要帮助助手做到三件事：在当前任务中读对材料，在长期合作中记住有效经验，在知识积累中保留清晰证据。

## 11. 参考资料

以下优先使用 Hermes、Hindsight、相关模型与实现项目的一手资料。

1. [Hermes Agent：快速入门](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart/)
2. [Hermes Agent：Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/)
3. [Hermes Agent：Gateway 授权与安全](https://hermes-agent.nousresearch.com/docs/user-guide/security)
4. [Hermes Agent：Skills 系统](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills/)
5. [Hermes Agent：使用 Skills](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills)
6. [Hermes Agent：MCP](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)
7. [Hermes Agent：MCP 使用指南](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes/)
8. [Hermes Agent：记忆提供方与 Hindsight 接入](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)
9. [Hindsight：Cross-Encoder 重排序技术说明](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory)
10. [Hindsight：部署说明](https://hindsight.vectorize.io/developer/installation)
11. [Sentence Transformers：Bi-Encoder 与 Cross-Encoder](https://www.sbert.net/examples/cross_encoder/applications/README.html)
12. [Sentence Transformers：检索与重排序](https://www.sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)
13. [Sentence Transformers：Cross-Encoder 使用说明](https://sbert.net/docs/cross_encoder/usage/usage.html)
14. [BAAI：BGE-M3 模型说明](https://huggingface.co/BAAI/bge-m3)
15. [hermes-lcm：项目与配置说明](https://github.com/stephenschoettler/hermes-lcm)
16. [Hermes Agent：LLM Wiki Skill](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/research/research-llm-wiki)
17. [pgvector：向量存储与相似度检索](https://github.com/pgvector/pgvector)
18. [Hermes Agent：SSH backend 配置](https://hermes-agent.nousresearch.com/docs/user-guide/configuration/#ssh-backend)
19. [Hermes Agent：CLI Shell 模式](https://hermes-agent.nousresearch.com/docs/user-guide/cli)
20. [OpenSSH：SSH 连接与认证](https://man.openbsd.org/ssh)
21. [OpenSSH：客户端配置](https://man.openbsd.org/ssh_config)
22. [Hindsight：重排序配置参考](https://github.com/vectorize-io/hindsight/blob/main/skills/hindsight-docs/references/developer/configuration.md)
23. [Hindsight：Cross-Encoder 实现](https://github.com/vectorize-io/hindsight/blob/main/hindsight-api-slim/hindsight_api/engine/cross_encoder.py)
24. [Hindsight：环境配置示例](https://github.com/vectorize-io/hindsight/blob/main/.env.example)
25. [Sentence Transformers：MS MARCO MiniLM 重排序模型卡](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2)
26. [Hindsight：Retain 的事实提取与向量存储](https://hindsight.vectorize.io/developer/retain)
27. [Hindsight：Retain 与 Recall 入门示例](https://hindsight.vectorize.io/cookbook/recipes/quickstart)
28. [Hindsight：多语言 Embedding 与 Rerank 配置](https://hindsight.vectorize.io/developer/multilingual)
29. [Hindsight：Recall 检索策略与 API 参考](https://github.com/vectorize-io/hindsight/blob/main/skills/hindsight-docs/references/developer/api/recall.md)
