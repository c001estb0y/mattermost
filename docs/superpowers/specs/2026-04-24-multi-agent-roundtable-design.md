# Multi-Agent Roundtable Design

Date: 2026-04-24
Updated: 2026-04-25

## Naming

User-facing name: **multi-agent roundtable**.

Internal implementation name: **cross-instance roundtable** or `cross_roundtable`.

`cross_roundtable` means the discussion crosses tunaPi process boundaries: each agent is a separate Mattermost bot backed by a separate tunaPi instance. This is different from the existing tunaPi `roundtable`, which is single-instance and calls multiple engines internally.

## Problem

Mattermost 频道中部署了多个 AI agent（如 `@kaixing`、`@agent2`），需要它们能在同一个 Thread 内互相 `@mention` 讨论，同时满足：

1. 防止无限递归对话
2. 人类可随时参与讨论或旁观
3. 人类之间的主频道闲聊不影响 agent 讨论
4. 讨论可暂停、恢复、结束
5. 任一 tunaPi 实例重启后，可以从 Mattermost Thread 历史恢复会议状态

## Key Decision

**Mattermost Thread 历史是共享事实来源。**

不要依赖某个 tunaPi 实例本地 JSON 作为跨实例状态的权威数据。两个 tunaPi 进程有各自的配置目录和进程内存，如果只把状态写到 starter 实例的本地文件，另一个实例无法可靠感知 pause/resume/close、轮次和参与者。

设计原则：

- Thread root post 保存会议元数据，或 root/thread 内第一条系统消息保存可解析的会议元数据。
- Thread 内系统消息记录控制事件：start、pause、resume、close、max_rounds reached。
- 每次收到相关 Thread 消息时，tunaPi 可通过 Mattermost API 拉取 Thread 历史并推导当前状态。
- 本地 store 只能作为缓存和加速层，不能作为跨实例事实来源。

## Architecture

### Thread 即会议室

利用 Mattermost 原生 Thread 作为讨论隔离空间，不自建会议服务。

```text
主频道 (agent-mem)
│
├── 人类闲聊...                         ← 不影响会议
├── 人类闲聊...
│
├── [根消息] 🎯 圆桌会议已开启           ← !rt start 创建
│   └── Thread (会议室)
│       ├── [system] metadata: participants, topic, max_rounds
│       ├── kaixing: 分析结果... @agent2
│       ├── agent2: 后端部分... @kaixing
│       ├── minusjiang: 别忘了鉴权       ← 人类直接回复
│       ├── kaixing: 好的...
│       ├── [system] paused
│       ├── [system] resumed
│       ├── agent2: 继续刚才的...
│       └── [system] closed
│
├── 人类继续闲聊...                      ← 完全不受影响
```

### 部署架构

两个独立 tunaPi 实例，各使用不同的 Mattermost Bot token：

| 实例 | Bot 名称 | 引擎 | 配置目录 | Persona |
|------|---------|------|----------|---------|
| tunaPi #1 | `@kaixing` | `codex` | `~/.tunapi` | 前端方向 |
| tunaPi #2 | `@agent2` | `codex` | `~/.tunapi-agent2` | 后端方向 |

每个实例忽略自己的消息，但会处理对方 bot 在 Thread 内对自己的 `@mention`。

## Repository Boundaries

本功能涉及三个仓库，但不应三仓库同时改代码。

| 仓库 | 角色 | 开发策略 |
|------|------|----------|
| `tunaPi` | 主功能实现 | 开 `feat/multi-agent-roundtable` 分支，完成命令、状态推导、Thread 路由、prompt 和测试 |
| `mattermost` | 部署编排入口 | 开 `feat/multi-agent-roundtable-deploy` 分支，放一键部署脚本、配置模板、systemd user service、验证脚本 |
| `codex` | 运行时依赖 | 先不改，不开功能分支；只在部署文档中固定版本和 PATH |

只有哪个仓库有实际变更，才在哪个仓库 commit/push。`codex` 只有在 tunaPi 集成验证证明 runner 能力不够时，才单独开分支。

## Session Lifecycle

### 状态机

```text
  !rt start ─────► [active] ◄──── !rt resume
                      │                 ↑
                达到轮次上限        根据 Thread 历史
                / !rt stop         继续构造下一轮 prompt
                / timeout
                      │                 ↑
                      ▼                 │
                   [paused] ────────────┘
                      │
                 !rt close
                      ▼
                   [closed]
```

### 状态说明

| 状态 | 事实来源 | agent 行为 |
|------|----------|------------|
| `active` | 最近控制事件不是 pause/close，且未达到轮次上限 | 被 `@mention` 的 bot 处理消息并回复 |
| `paused` | Thread 内最近控制事件为 pause 或超时自动 pause | bot 不继续发言，人类可继续回复 |
| `closed` | Thread 内存在 close 或 max_rounds reached 事件 | bot 不再参与该 Thread |

## Command System

所有控制指令走 `!rt` 前缀，由 tunaPi 命令层解析，不经过 LLM。

### 命令列表

| 命令 | 前置条件 | 行为 |
|------|----------|------|
| `!rt start @A @B 主题` | 当前频道无 active 会议 | 创建 root post，写入会议元数据，开启 Thread |
| `!rt stop` | Thread 内，有 active 会议 | 在 Thread 写入 pause 控制事件 |
| `!rt resume` | Thread 内，有 paused 会议 | 在 Thread 写入 resume 控制事件，并提示下一位 agent |
| `!rt close` | Thread 内，有 active/paused 会议 | 在 Thread 写入 close 控制事件 |
| `!rt status` | 任意 | 从 Thread 或频道最近 root posts 推导状态 |

`!rt list` 和 `!rt config` 不进入首版。首版只做 start/stop/resume/close/status，降低实现面。

## Start Flow

```text
1. 人类输入:
   !rt start @kaixing @agent2 分析 codebuddy-mem 架构

2. 接收到命令的 tunaPi 创建主频道 root post:
   🎯 圆桌会议已开启
   主题: 分析 codebuddy-mem 架构
   参与 Agent: @kaixing @agent2
   轮次上限: 3

3. tunaPi 在 root thread 内写入可解析的 metadata/control 消息。

4. tunaPi 在 thread 内 @mention 第一位 agent：
   @kaixing 圆桌讨论开始。请发表观点，完成后 @agent2。

5. @kaixing 所在 tunaPi 收到 mention，拉取 Thread 历史，构造 roundtable prompt，调用 codex。

6. @kaixing 回复并 @agent2。

7. @agent2 所在 tunaPi 重复同样流程。

8. 任一实例发现已达到 max_rounds 或收到 close，写入 closed 控制事件并停止。
```

## Message Handling Rules

### Thread 内消息处理

| 消息来源 | 处理方式 | 是否计入 round |
|----------|----------|----------------|
| bot participant `@mention` 当前 bot | 拉取 Thread 历史，推导状态，构造 prompt，调用 engine | 是 |
| 人类普通回复 | 保留在 Thread，下一轮 prompt 纳入上下文 | 否 |
| `!rt stop/resume/close/status` | 写入或读取控制事件 | 否 |
| 非参与 bot 或 closed Thread | 忽略 | 否 |

### 主频道消息处理

| 消息来源 | 处理方式 |
|----------|----------|
| 人类闲聊（不 `@bot`） | 忽略，不影响会议 |
| 人类 `@bot` | 走现有 trigger_mode 逻辑，与会议无关 |
| `!rt start` | 如果频道已有 active 会议，拒绝并提示现有 Thread |

## Anti-Loop Mechanism

### 硬性上限

- 配置 `max_rounds`，默认 3。
- 一轮定义为每个参与 bot 完成一次发言；实现时通过 Thread 历史中的 participant 发言序列推导。
- 达到上限后写入 closed 控制事件。

### AI 自主终止

prompt 明确要求：

> 如果你认为讨论已达成共识或无新信息可补充，可以不 `@` 下一位 agent，直接总结结论。

agent 不 `@` 下一位参与者时，不会触发下一轮。

### 超时保护

- active 会议超过 `timeout_minutes` 无新 participant 发言时，任一实例在下次检查时写入 paused 控制事件。
- 不需要后台常驻扫描器作为首版要求；首版可采用“收到相关消息时懒检查”。

## Persistence And Recovery

### 权威状态

权威状态来自 Mattermost：

```text
GET /api/v4/posts/{root_post_id}/thread
```

从 Thread 历史推导：

- participants
- topic
- max_rounds
- current_round
- active/paused/closed
- next participant
- human comments for prompt context

### 本地缓存

每个 tunaPi 实例可以维护本地缓存：

```text
~/.tunapi/cross_roundtable_cache.json
~/.tunapi-agent2/cross_roundtable_cache.json
```

缓存只用于减少 API 调用。缓存缺失、过期或与 Thread 历史冲突时，以 Thread 历史为准。

## Configuration

配置位于 Mattermost transport 下，因为该功能依赖 Mattermost Thread 和 `@mention`：

```toml
[transports.mattermost.cross_roundtable]
enabled = true
max_rounds = 3
timeout_minutes = 5
```

部署侧使用两份 tunaPi 配置：

```text
~/.tunapi/tunapi.toml
~/.tunapi-agent2/tunapi.toml
```

## Prompt Design

### Agent 参与圆桌时的 prompt 附加段

```text
你正在参与一场 Mattermost multi-agent roundtable。

规则：
1. 你是参与者之一，当前 Thread 是会议室。
2. 阅读 Thread 历史，包括其他 agent 和人类的发言。
3. 发言保持简洁，默认不超过 300 字。
4. 如果需要下一位 agent 回应，请在结尾明确 @对方用户名。
5. 如果讨论已收敛或无新信息，不要 @下一位，直接总结结论。
6. 不要伪造控制命令；暂停、恢复、结束只由 !rt 控制。
```

## Development And Deployment Scope

### tunaPi 需要修改

1. `src/tunapi/core/cross_roundtable.py`：Thread metadata、状态推导、轮次计算、prompt builder。
2. `src/tunapi/mattermost/commands.py`：`!rt start/stop/resume/close/status` 解析。
3. `src/tunapi/mattermost/loop.py`：Thread 路由、cross_roundtable 调度、与现有 `!rt` 共存。
4. `src/tunapi/mattermost/parsing.py` 和 `types.py`：保留 `root_id`、`sender_username`，按需补充 bot 来源识别。
5. `src/tunapi/settings.py`、`mattermost/backend.py`、`mattermost/bridge.py`：传递 cross_roundtable 配置。
6. tests：覆盖状态推导、命令解析、Thread transcript prompt、路由条件。

### mattermost 仓库负责部署编排

1. `deploy/roundtable/deploy.ps1`：Windows 本地一键部署入口。
2. `deploy/roundtable/deploy.sh`：远端 Linux 部署逻辑。
3. `deploy/roundtable/templates/`：两份 tunaPi 配置模板和 systemd user service 模板。
4. `deploy/roundtable/verify.sh`：验证两个 tunaPi 进程、heartbeat、日志和 Mattermost bot 可用性。

### codex 暂不修改

codex 作为 tunaPi runner 依赖。首版只固定 PATH 和版本，不改 codex 仓库。

### 不需要改

- Mattermost 服务端源码：无需改。
- 现有 tunaPi 单实例 `!rt "topic"`：保留，不破坏。
- GitHub 插件：不受影响。
