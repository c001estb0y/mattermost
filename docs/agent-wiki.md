# Agent 使用指南

本文档汇总 tunaPi Mattermost Bot 的 Agent 工作区管理和多 Agent 圆桌会议相关命令。

---

## 1. 架构概览

```
agent-runtime/
├── agent-envs/          # Agent 自身环境（Git 管理）
│   ├── kaixing/         #   AGENTS.md, .codex/, .agents/skills/ ...
│   └── codeview/
├── runtime-homes/       # 运行时 HOME（启动前自动从 agent-envs 映射）
│   ├── kaixing/         #   .codex/.env, config.toml, rules, skills ...
│   └── codeview/
├── workspaces/          # 干活工作区（可多个，Git 管理）
│   ├── agent-mem/
│   ├── kaixing-workspace/
│   └── codeview-workspace/
├── channels/            # 频道绑定配置（自动生成）
│   └── mattermost-<channel_id>/bindings.toml
├── runs/                # 运行清单记录
├── locks/               # 工作区写入锁
└── logs/
```

### 三层模型

| 层 | 说明 | 隔离方式 |
|---|------|---------|
| Agent 自身环境 | 每个 Agent 独有的人设、规则、技能、API Key | 独立 `HOME` 目录 |
| Channel 工作区绑定 | 频道级别指定哪个 Agent 用哪个工作区 | 频道 TOML 配置 |
| Active Workspace | Agent 实际干活的代码目录（`cwd`） | 可共享，通过 workspace lock 串行写入 |

### 环境变量（Codex 子进程可读）

| 变量 | 含义 |
|------|------|
| `HOME` | Agent 运行时家目录（`runtime-homes/<agent>`） |
| `TUNAPI_AGENT_ID` | 当前 Agent 标识 |
| `TUNAPI_AGENT_ENV_DIR` | Agent 自身环境源目录 |
| `TUNAPI_WORKSPACE_DIR` | 绑定的工作区路径 |
| `TUNAPI_ACTIVE_WORKSPACE_NAME` | 工作区名称 |
| `TUNAPI_CHANNEL_CONTEXT_DIR` | 频道上下文目录（可选） |

---

## 2. Workspace 管理命令

在 Mattermost 频道中发送以下命令。注意：**每条命令单独发一条消息**，等 bot 回复后再发下一条。

### `!workspace add <name> <path> [repo]`

添加一个工作区到当前频道。

```
!workspace add agent-mem /data/home/minusjiang/agent-runtime/workspaces/agent-mem
```

- `name`：工作区名称（用于后续引用）
- `path`：CVM 上的绝对路径
- `repo`：可选，关联的 Git 仓库 URL

### `!workspace use <name>`

设置频道默认工作区。未绑定特定 agent 的请求会使用这个。

```
!workspace use agent-mem
```

### `!workspace bind <agent> <workspace>`

将某个 Agent 绑定到指定工作区。绑定后该 Agent 的 `cwd` 会自动切到这个目录。

```
!workspace bind kaixing agent-mem
!workspace bind codeview agent-mem
```

### `!workspace info`

查看当前 bot 在本频道的活跃工作区。

```
!workspace info
```

### `!workspace list`

列出当前频道的所有工作区和 Agent 绑定关系。

```
!workspace list
```

输出示例：
```
Channel workspaces

Workspaces:
• agent-mem path: /data/home/minusjiang/agent-runtime/workspaces/agent-mem

Agent bindings:
• codeview -> agent-mem
• kaixing -> agent-mem

Default workspace: agent-mem
```

---

## 3. 典型 Workspace 设置流程

```
!workspace add agent-mem /data/home/minusjiang/agent-runtime/workspaces/agent-mem
!workspace use agent-mem
!workspace bind kaixing agent-mem
!workspace bind codeview agent-mem
!workspace list
```

设置完成后，`@kaixing` 和 `@codeview` 的后续任务都会在 `agent-mem` 目录下执行。

---

## 4. 圆桌会议命令

多 Agent 圆桌让两个或多个 bot 在同一个 Thread 中围绕一个话题结构化讨论。

### `!rt start @bot1 @bot2 <topic>`

启动圆桌讨论。

```
!rt start @kaixing @codeview 讨论 agent-mem 仓库的 code review 流程
```

- 至少需要 2 个 `@bot` 参与者
- `topic` 是讨论主题，必填
- 发起后 bot 会在 Thread 中轮流发言

### `!rt stop`

暂停当前圆桌（保留状态，可恢复）。

```
!rt stop
```

### `!rt resume`

恢复已暂停的圆桌讨论。

```
!rt resume
```

### `!rt close`

关闭圆桌讨论（终止生命周期）。

```
!rt close
```

### `!rt status`

查看当前圆桌状态。

```
!rt status
```

### 圆桌结束后的 Thread 行为

圆桌生命周期结束后，在同一个 Thread 中 `@bot` 仍然可以正常聊天。bot 会：
- 读取 Thread 历史作为上下文
- 以普通聊天模式回复（不再轮流发言）
- 不会触发新的圆桌循环

---

## 5. 其他常用命令

| 命令 | 说明 |
|------|------|
| `!help` | 显示所有可用命令 |
| `!new` | 开始新的对话会话 |
| `!model <engine> [model]` | 切换引擎或模型 |
| `!models [engine]` | 列出可用模型 |
| `!project list\|set\|info` | 管理项目绑定 |
| `!status` | 当前会话状态 |
| `!cancel` | 取消正在运行的任务 |

---

## 6. 已知限制 & 后续优化

1. **双 bot 重复响应管理命令**：`!workspace` 等管理命令两个 bot 都会处理，一个成功一个报 locked。后续将优化为仅 owner bot 响应。
2. **`.codex/.env` 需手动放到 agent-envs**：当前需要手动将 API Key 放入 `agent-envs/<agent>/.codex/.env`，后续会自动化。
3. **workspace lock 超时**：如果 bot 进程异常退出，lock 可能残留；系统支持基于 PID 的 stale lock 自动恢复。
4. **非容器级隔离**：当前隔离是进程级 HOME/cwd，不是 Docker/VM 级别。共享 workspace 的写入通过串行锁保证一致性。
