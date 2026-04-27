# Roundtable Thread Continuation Design

Date: 2026-04-27

## Problem

multi-agent roundtable 使用 Mattermost Thread 作为共享事实来源。圆桌处于 active 状态时，这个 Thread 应由圆桌状态机接管，保证 `@kaixing` 和 `@codeview` 按顺序发言，并避免 bot 之间互相触发导致循环。

但圆桌自然结束后，同一个 Thread 仍然应该可以继续作为普通讨论 Thread 使用。用户应该能在这个 Thread 里继续 `@bot`，让 bot 基于刚刚的圆桌内容总结方案、补充细节或回答追问。

当前行为不够理想：只要 Thread 中存在 roundtable metadata，即使已经达到 `max_rounds`，也可能继续被 roundtable router 识别为圆桌 Thread。此时用户在 Thread 内 `@bot` 的普通追问可能被圆桌路由吞掉，导致没有响应。

## Goals

1. 圆桌 active 时，保留严格的 roundtable 路由和 anti-loop 规则。
2. bot 正在生成圆桌发言时，外部对同一个 bot 的普通 `@mention` 应收到忙碌提示，而不是启动另一个 engine run。
3. 圆桌不再 active 后，同一 Thread 内的人类 `@bot` 应恢复为普通 Thread 聊天。
4. 圆桌后的普通 Thread 聊天必须把前面的圆桌讨论注入 prompt context。
5. 圆桌结束后的 bot-to-bot mention 不应重新触发自动圆桌循环。
6. `!rt resume` 是显式恢复圆桌模式的入口，且只在状态允许时生效。

## Non-Goals

- 不实现持久化全局任务队列来延迟处理外部 mention。
- 不把所有 bot 全局串行化；忙碌状态只作用于当前 bot 实例。
- 不引入新的数据库保存 roundtable 状态；Mattermost Thread 历史仍是事实来源。
- 不改变现有 `!rt start @A @B topic` 命令形态。

## Recommended Approach

采用「进程内 roundtable busy flag」加「更明确的 Thread mode routing」。

每个 tunaPi 进程代表一个 Mattermost bot。当这个进程运行 `_run_cross_roundtable_engine` 时，将当前 bot 标记为 busy。若同一个 bot 在 busy 期间收到外部普通 mention，tunaPi 回复一条短忙碌提示，并且不启动新的 engine run。

同时，cross-roundtable router 需要区分 active 圆桌模式和 post-roundtable 普通聊天模式：

- `active`：roundtable router 接管 Thread，只处理参与者 bot 的有序发言。
- `paused`、`closed`、`max_rounds reached`：人类 mention 释放给普通 Mattermost chat path。
- 非 active 圆桌 Thread 内的 bot-authored message 继续被忽略，避免重新触发循环。

## User-Facing Behavior

### Active Roundtable

如果 `kaixing` 正在生成圆桌回复，而用户在圆桌 Thread 外 `@kaixing`，tunaPi 应回复：

```text
@kaixing 正在参与圆桌讨论中，当前不接受新的外部请求。请稍后再试，或在圆桌 Thread 内继续讨论。
```

其中 bot 名称应与被 mention 的 bot 保持一致。

忙碌提示适用于：

- 主频道 mention。
- 其他 Thread 内的 mention。
- 所有原本会进入普通聊天的 mention。

当 bot 只是等待另一个参与者发言时，不应视为 busy。等待不是生成中。

### Post-Roundtable Chat

圆桌达到 `max_rounds`、收到 `!rt close`，或推导出其他非 active 状态后，用户可以继续在同一 Thread 内讨论：

```text
@kaixing 总结一下上述讨论之后的实现方案
```

预期行为：

1. 这条消息不会被 roundtable router 消费。
2. 消息进入普通 Mattermost chat path。
3. bot 在同一 Thread 内回复。
4. prompt 包含这个 Thread 中此前的圆桌讨论内容。

圆桌非 active 后，bot-authored message 即使 mention 了另一个 bot，也不应触发普通聊天。这样可以防止旧 bot 消息或最终总结重新启动循环。

### Explicit Resume

`!rt resume` 仍然是把 paused 圆桌恢复为 active 圆桌模式的显式入口。圆桌结束后的普通人类 `@bot` 不会隐式恢复圆桌。

## Thread Context Injection

当普通聊天发生在 Thread 内时，Mattermost path 应通过 `get_thread(root_id)` 拉取 Thread 历史，并把一段紧凑的 context block 前置到 prompt。

如果 Thread 包含 roundtable metadata，context block 应包含：

- 圆桌 topic。
- 参与者列表。
- 推导出的当前状态。
- 最近的人类消息。
- bot 的讨论消息。
- 当前请求。

`working · codex`、`starting · codex`、只包含 resume token 的短消息等状态/进度消息，应尽量从上下文中省略。

建议格式：

```text
[Thread context]
This Mattermost Thread previously contained a multi-agent roundtable.
Topic: 讨论一下我如果要实现一个计算器，应该怎么做？
Participants: kaixing, codeview
Status: completed

Recent Thread messages:
[kaixing]: ...
[codeview]: ...
[minusjiang]: @kaixing 总结一下上述讨论之后的实现方案

[Current request]
总结一下上述讨论之后的实现方案
```

实现应限制上下文大小，避免 prompt 过长。第一版可采用最近 20 条有意义 Thread posts，或约 12,000 字符上限，同时始终保留 root metadata 和 topic。

## State and Routing Rules

### Roundtable Active

cross-roundtable router 只有在以下条件全部成立时才处理 Thread 消息：

1. Thread 包含合法 roundtable metadata。
2. 推导出的 roundtable state 是 `active`。
3. 发送者是参与者 bot。
4. 消息 mention 了当前 bot。
5. 当前 bot 是 expected next participant。

active 圆桌 Thread 内的人类消息会保留为 Thread 历史。除非是合法的 `!rt` 控制命令，否则不应立即触发 bot turn。

### Roundtable Non-Active

当 Thread 包含 roundtable metadata，但推导出的状态不是 `active` 时：

- `!rt status`、`!rt close`、合法的 `!rt resume` 仍走 command/control handling。
- 人类 `@bot` 消息应让 cross-roundtable router 返回 `False`，由普通 chat path 继续处理。
- bot-authored message 应返回 `True` 并被忽略，不进入普通 chat path。

## Busy State

busy state 只存在于当前 tunaPi 进程内，并且作用域是当前 bot identity。

进程在启动 cross-roundtable engine run 前立即标记 busy，并在 run 完成、失败或取消后的 `finally` 中清除 busy。

普通 chat path 在 command handling 之后、调用 `_run_engine` 之前检查 busy state。如果 busy 已设置，且当前消息不属于正在处理的 active roundtable turn，则发送忙碌提示，不启动新的 run。

这意味着：

- 只有 `kaixing` 进程正在生成圆桌发言时，`@kaixing` 才会被阻塞。
- 如果 `codeview` 进程不 busy，`@codeview` 仍可响应。
- 进程重启会清除本地 busy state。未完成 runner work 继续由现有 lifecycle 和 pending-run 机制处理。

## Error Handling

- 如果 post-roundtable 普通聊天时 `get_thread(root_id)` 失败，bot 仍应回答当前消息，但可短句说明 Thread 历史加载失败。
- 如果 Thread 历史过大，应使用确定性截断，并保留最近的有意义 posts。
- 如果 metadata 格式损坏，应退回普通 Thread context，不包含 roundtable-specific topic/status 字段。
- 如果忙碌提示发送失败，应记录日志，并且仍不启动外部 engine run。

## Tests

围绕 Mattermost loop routing 增加聚焦测试：

1. active 圆桌期间，外部 mention 同一个 busy bot 会发送忙碌提示，并且不会调用 `_run_engine`。
2. active 圆桌期间，外部 mention 另一个 idle bot 仍可进入普通 `_run_engine`。
3. active 圆桌 Thread 内的人类评论会作为历史保留，但不立即触发 bot turn。
4. 达到 `max_rounds` 后，同一 Thread 内的人类 `@kaixing` 会释放给普通聊天。
5. post-roundtable Thread 的普通聊天 prompt 包含 topic、participants、此前 bot 讨论和当前请求。
6. 达到 `max_rounds` 后，包含 `@kaixing` 的 bot-authored message 会被忽略，不调用普通 `_run_engine`。
7. 可恢复 paused Thread 上的 `!rt resume` 会重新进入 active roundtable routing。
8. `_run_cross_roundtable_engine` 抛错时 busy state 会被清除。

## Acceptance Criteria

- 用户可以运行一次 multi-agent roundtable 到完成。
- 完成后，用户可以在同一 Thread 内继续 `@kaixing` 追问。
- 追问回复会使用此前圆桌讨论作为上下文。
- 外部 mention 正在生成圆桌发言的 bot 时，会得到忙碌提示。
- 圆桌结束后不会引入新的 bot-to-bot loop。
