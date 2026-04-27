# Roundtable Thread Continuation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a Mattermost roundtable Thread return to normal human-to-bot chat after the roundtable is no longer active, while preserving roundtable context and preventing bot loops.

**Architecture:** Mattermost Thread history remains the source of truth. `tunaPi` keeps the active roundtable router strict, releases non-active human Thread mentions to normal chat, injects compact Thread context into ordinary prompts, and uses a per-process busy flag to reject external mentions while the current bot is generating a roundtable turn.

**Tech Stack:** Python 3.14, tunaPi, Mattermost WebSocket/API, Mattermost Thread replies, `pytest`, `anyio`, `unittest.mock`.

---

## Scope Check

This plan only changes `tunaPi`. It does not modify Mattermost server code, `codex`, deployment scripts, or bot account provisioning.

The design spec is `docs/superpowers/specs/2026-04-27-roundtable-thread-continuation-design.md`.

---

## File Structure

### tunaPi

| File | Action | Responsibility |
|------|--------|----------------|
| `src/tunapi/core/cross_roundtable.py` | Modify | Add transport-neutral helpers for meaningful Thread posts and post-roundtable context prompt formatting |
| `src/tunapi/mattermost/loop.py` | Modify | Release non-active human mentions to normal chat, inject Thread context, add roundtable busy state |
| `tests/test_cross_roundtable.py` | Modify | Unit tests for context prompt formatting and status/progress filtering |
| `tests/test_mattermost_cross_rt_loop.py` | Modify | Routing tests for non-active Thread release and bot-authored ignore behavior |
| `tests/test_mm_loop_extra.py` | Modify | Dispatch-level tests for context injection and busy state |

Do not change `MattermostIncomingMessage` fields for this feature. Existing `sender_username`, `sender_id`, `root_id`, and `text` are sufficient.

---

## Phase 0: Baseline

### Task 0.1: Confirm working branch and focused baseline

**Files:** none

- [ ] **Step 1: Go to the tunaPi feature worktree**

Run:

```powershell
Set-Location e:\Github\mattermost\tunaPi\.worktrees\multi-agent-roundtable
git status -sb
```

Expected: branch is the multi-agent roundtable feature branch, and there are no unrelated unstaged edits that would be overwritten.

- [ ] **Step 2: Run current focused tests**

Run:

```powershell
uv run pytest tests/test_cross_roundtable.py tests/test_mattermost_cross_rt_loop.py tests/test_mm_loop_extra.py -v --no-cov
```

Expected: tests pass, or any failure is recorded as pre-existing before this feature work begins.

---

## Phase 1: Core Thread Context Formatting

### Task 1.1: Add failing core tests for post-roundtable context

**Files:**
- Modify: `tests/test_cross_roundtable.py`

- [ ] **Step 1: Add imports**

Add these imports near the existing `cross_roundtable` imports:

```python
from tunapi.core.cross_roundtable import (
    CrossRTMetadata,
    CrossRTState,
    CrossRTStatus,
    ThreadPost,
    build_thread_context_prompt,
    meaningful_thread_posts,
)
```

If the file already imports some of these names, merge the import list instead of creating duplicate imports.

- [ ] **Step 2: Add tests**

Append these tests to `tests/test_cross_roundtable.py`:

```python
def test_meaningful_thread_posts_filters_roundtable_system_noise():
    posts = [
        ThreadPost(
            sender_username="minusjiang",
            message='<!-- tunapi:roundtable {"version":1,"topic":"计算器","participants":["kaixing","codeview"],"max_rounds":1} -->',
            created_at=1,
            root_id="root1",
        ),
        ThreadPost(
            sender_username="kaixing",
            message="working · codex",
            created_at=2,
            root_id="root1",
        ),
        ThreadPost(
            sender_username="kaixing",
            message="可以先实现加减乘除 @codeview",
            created_at=3,
            root_id="root1",
        ),
        ThreadPost(
            sender_username="codeview",
            message="后端建议补充输入校验",
            created_at=4,
            root_id="root1",
        ),
    ]

    result = meaningful_thread_posts(posts)

    assert [post.message for post in result] == [
        "可以先实现加减乘除 @codeview",
        "后端建议补充输入校验",
    ]


def test_build_thread_context_prompt_includes_roundtable_summary_fields():
    metadata = CrossRTMetadata(
        topic="讨论计算器实现",
        participants=["kaixing", "codeview"],
        max_rounds=1,
    )
    state = CrossRTState(
        metadata=metadata,
        status=CrossRTStatus.CLOSED,
        current_round=1,
        next_participant=None,
    )
    posts = [
        ThreadPost(
            sender_username="minusjiang",
            message="讨论计算器实现",
            created_at=1,
            root_id="root1",
        ),
        ThreadPost(
            sender_username="kaixing",
            message="前端用一个表单和结果区 @codeview",
            created_at=2,
            root_id="root1",
        ),
        ThreadPost(
            sender_username="codeview",
            message="后端只需要纯函数和单元测试",
            created_at=3,
            root_id="root1",
        ),
    ]

    prompt = build_thread_context_prompt(
        posts=posts,
        current_request="总结一下上述讨论",
        metadata=metadata,
        state=state,
        max_posts=20,
        max_chars=12_000,
    )

    assert "Topic: 讨论计算器实现" in prompt
    assert "Participants: kaixing, codeview" in prompt
    assert "Status: closed" in prompt
    assert "[kaixing]: 前端用一个表单和结果区 @codeview" in prompt
    assert "[codeview]: 后端只需要纯函数和单元测试" in prompt
    assert "[Current request]\n总结一下上述讨论" in prompt
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```powershell
uv run pytest tests/test_cross_roundtable.py -v --no-cov
```

Expected: failure because `meaningful_thread_posts` and `build_thread_context_prompt` are not defined.

### Task 1.2: Implement core context helpers

**Files:**
- Modify: `src/tunapi/core/cross_roundtable.py`

- [ ] **Step 1: Add progress-message filter helpers**

Add this code after `is_system_marker_post`:

```python
_PROGRESS_MESSAGE_RE = re.compile(
    r"^\s*(working|starting|thinking|running)\s*[·.-]\s*\w+\s*$",
    re.IGNORECASE,
)


def is_progress_marker_post(text: str) -> bool:
    stripped = text.strip()
    if not stripped:
        return True
    if _PROGRESS_MESSAGE_RE.match(stripped):
        return True
    if stripped.lower().startswith("resume token:"):
        return True
    return False


def meaningful_thread_posts(posts: list[ThreadPost]) -> list[ThreadPost]:
    result: list[ThreadPost] = []
    for post in sorted(posts, key=lambda p: p.created_at):
        if is_system_marker_post(post.message):
            continue
        if is_progress_marker_post(post.message):
            continue
        result.append(post)
    return result
```

- [ ] **Step 2: Add context prompt builder**

Add this code after `build_agent_prompt`:

```python
def build_thread_context_prompt(
    *,
    posts: list[ThreadPost],
    current_request: str,
    metadata: CrossRTMetadata | None = None,
    state: CrossRTState | None = None,
    max_posts: int = 20,
    max_chars: int = 12_000,
) -> str:
    meaningful = meaningful_thread_posts(posts)
    selected = meaningful[-max_posts:] if max_posts > 0 else meaningful

    header_lines = ["[Thread context]"]
    if metadata is not None:
        header_lines.append(
            "This Mattermost Thread previously contained a multi-agent roundtable."
        )
        header_lines.append(f"Topic: {metadata.topic}")
        header_lines.append(f"Participants: {', '.join(metadata.participants)}")
    if state is not None:
        header_lines.append(f"Status: {state.status.value}")

    message_lines = ["", "Recent Thread messages:"]
    for post in selected:
        sender = post.sender_username.lstrip("@") or "unknown"
        message_lines.append(f"[{sender}]: {post.message.strip()}")

    result = "\n".join(
        [
            *header_lines,
            *message_lines,
            "",
            "[Current request]",
            current_request.strip(),
        ]
    )

    if len(result) <= max_chars:
        return result

    truncated_messages: list[str] = []
    budget = max_chars - len("\n".join(header_lines)) - len(current_request) - 80
    used = 0
    for line in reversed(message_lines[2:]):
        line_length = len(line) + 1
        if used + line_length > max(budget, 0):
            break
        truncated_messages.append(line)
        used += line_length
    truncated_messages.reverse()

    return "\n".join(
        [
            *header_lines,
            "",
            "Recent Thread messages:",
            *truncated_messages,
            "",
            "[Current request]",
            current_request.strip(),
        ]
    )
```

- [ ] **Step 3: Run core tests**

Run:

```powershell
uv run pytest tests/test_cross_roundtable.py -v --no-cov
```

Expected: core tests pass.

- [ ] **Step 4: Commit core helpers**

Run:

```powershell
git add src/tunapi/core/cross_roundtable.py tests/test_cross_roundtable.py
@'
feat: add roundtable thread context helpers

Add transport-neutral helpers for filtering roundtable noise and building post-roundtable Thread context prompts.
'@ | git commit -F -
```

---

## Phase 2: Release Non-Active Human Thread Mentions

### Task 2.1: Add failing routing tests for non-active Threads

**Files:**
- Modify: `tests/test_mattermost_cross_rt_loop.py`

- [ ] **Step 1: Add closed Thread human mention test**

Append this test:

```python
@pytest.mark.anyio
async def test_closed_roundtable_human_mention_is_released_to_normal_chat():
    cfg = _make_cfg(bot_username="kaixing")
    cfg.bot._client.get_thread = AsyncMock(
        return_value=PostList(
            order=["root1", "p1", "p2"],
            posts={
                "root1": Post(
                    id="root1",
                    channel_id="ch1",
                    user_id="u-human",
                    message='<!-- tunapi:roundtable {"version":1,"topic":"分析架构","participants":["kaixing","codeview"],"max_rounds":1} -->',
                ),
                "p1": Post(
                    id="p1",
                    channel_id="ch1",
                    user_id="u-kaixing",
                    root_id="root1",
                    message="前端视角 @codeview",
                    create_at=1,
                ),
                "p2": Post(
                    id="p2",
                    channel_id="ch1",
                    user_id="u-codeview",
                    root_id="root1",
                    message="后端视角",
                    create_at=2,
                ),
            },
        )
    )
    cfg.bot.get_user = AsyncMock(
        side_effect=lambda user_id: User(
            id=user_id,
            username={
                "u-human": "minusjiang",
                "u-kaixing": "kaixing",
                "u-codeview": "codeview",
            }[user_id],
            is_bot=user_id != "u-human",
        )
    )
    msg = _make_msg(
        "@kaixing 总结一下上述讨论",
        root_id="root1",
        sender_username="minusjiang",
    )

    result = await _try_dispatch_cross_roundtable(
        msg,
        cfg,
        {},
        MagicMock(),
        None,
        AsyncMock(side_effect=_send_capture),
    )

    assert result is False
```

- [ ] **Step 2: Add closed Thread bot mention ignore test**

Append this test:

```python
@pytest.mark.anyio
async def test_closed_roundtable_bot_mention_is_ignored():
    cfg = _make_cfg(bot_username="kaixing")
    cfg.bot._client.get_thread = AsyncMock(
        return_value=PostList(
            order=["root1", "p1", "p2"],
            posts={
                "root1": Post(
                    id="root1",
                    channel_id="ch1",
                    user_id="u-human",
                    message='<!-- tunapi:roundtable {"version":1,"topic":"分析架构","participants":["kaixing","codeview"],"max_rounds":1} -->',
                ),
                "p1": Post(
                    id="p1",
                    channel_id="ch1",
                    user_id="u-kaixing",
                    root_id="root1",
                    message="前端视角 @codeview",
                    create_at=1,
                ),
                "p2": Post(
                    id="p2",
                    channel_id="ch1",
                    user_id="u-codeview",
                    root_id="root1",
                    message="后端视角 @kaixing",
                    create_at=2,
                ),
            },
        )
    )
    cfg.bot.get_user = AsyncMock(
        side_effect=lambda user_id: User(
            id=user_id,
            username={
                "u-human": "minusjiang",
                "u-kaixing": "kaixing",
                "u-codeview": "codeview",
            }[user_id],
            is_bot=user_id != "u-human",
        )
    )
    msg = _make_msg(
        "后端视角 @kaixing",
        root_id="root1",
        sender_username="codeview",
    )

    with patch("tunapi.mattermost.loop._run_engine", new_callable=AsyncMock) as run:
        result = await _try_dispatch_cross_roundtable(
            msg,
            cfg,
            {},
            MagicMock(),
            None,
            AsyncMock(side_effect=_send_capture),
        )

    assert result is True
    run.assert_not_awaited()
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```powershell
uv run pytest tests/test_mattermost_cross_rt_loop.py -v --no-cov
```

Expected: the human release test fails because current code returns `True` for all non-active states.

### Task 2.2: Implement non-active routing release

**Files:**
- Modify: `src/tunapi/mattermost/loop.py`

- [ ] **Step 1: Replace non-active state branch**

In `_try_dispatch_cross_roundtable`, replace:

```python
    if state.status != CrossRTStatus.ACTIVE:
        return True
```

with:

```python
    if state.status != CrossRTStatus.ACTIVE:
        if await _sender_is_bot(msg, cfg):
            logger.info(
                "cross_roundtable.non_active_bot_message_ignored",
                status=state.status.value,
                sender=msg.sender_username,
                root_id=msg.root_id,
            )
            return True
        logger.info(
            "cross_roundtable.non_active_human_message_released",
            status=state.status.value,
            sender=msg.sender_username,
            root_id=msg.root_id,
        )
        return False
```

This branch is after `!rt` command handling, so `!rt status`, `!rt close`, and valid `!rt resume` remain controlled by the roundtable router.

- [ ] **Step 2: Run routing tests**

Run:

```powershell
uv run pytest tests/test_mattermost_cross_rt_loop.py -v --no-cov
```

Expected: routing tests pass.

- [ ] **Step 3: Commit routing change**

Run:

```powershell
git add src/tunapi/mattermost/loop.py tests/test_mattermost_cross_rt_loop.py
@'
fix: release completed roundtable threads to normal chat

Let human mentions in non-active roundtable Threads fall through to normal chat while continuing to ignore bot-authored messages.
'@ | git commit -F -
```

---

## Phase 3: Inject Thread Context Into Normal Chat

### Task 3.1: Add failing dispatch test for post-roundtable prompt context

**Files:**
- Modify: `tests/test_mm_loop_extra.py`

- [ ] **Step 1: Add imports**

Add these imports near the existing imports:

```python
from tunapi.mattermost.api_models import Post, PostList, User
```

If `User` is already imported in this file, do not duplicate it.

- [ ] **Step 2: Add test**

Append this test:

```python
@pytest.mark.anyio()
async def test_completed_roundtable_thread_human_mention_gets_thread_context():
    cfg = _make_cfg(bot_username="kaixing")
    cfg.cross_roundtable_enabled = True
    cfg.bot = MagicMock()
    cfg.bot._client = MagicMock()
    cfg.bot._client.get_thread = AsyncMock(
        return_value=PostList(
            order=["root1", "p1", "p2", "human1"],
            posts={
                "root1": Post(
                    id="root1",
                    channel_id="ch1",
                    user_id="u-human",
                    message='<!-- tunapi:roundtable {"version":1,"topic":"讨论计算器实现","participants":["kaixing","codeview"],"max_rounds":1} -->',
                ),
                "p1": Post(
                    id="p1",
                    channel_id="ch1",
                    user_id="u-kaixing",
                    root_id="root1",
                    message="前端使用表单和结果区 @codeview",
                    create_at=1,
                ),
                "p2": Post(
                    id="p2",
                    channel_id="ch1",
                    user_id="u-codeview",
                    root_id="root1",
                    message="后端使用纯函数和单元测试",
                    create_at=2,
                ),
                "human1": Post(
                    id="human1",
                    channel_id="ch1",
                    user_id="u-human",
                    root_id="root1",
                    message="@kaixing 总结一下上述讨论",
                    create_at=3,
                ),
            },
        )
    )
    cfg.bot.get_user = AsyncMock(
        side_effect=lambda user_id: User(
            id=user_id,
            username={
                "u-human": "minusjiang",
                "u-kaixing": "kaixing",
                "u-codeview": "codeview",
            }[user_id],
            is_bot=user_id != "u-human",
        )
    )
    cfg.runtime.resolve_message.return_value = MagicMock(
        context=MagicMock(project=None),
        engine_override=None,
    )
    cfg.runtime.resolve_engine.return_value = MagicMock(name="codex")
    cfg.runtime.format_context_line.return_value = ""
    cfg.runtime.resolve_run_cwd.return_value = None

    msg = _make_msg(
        text="@kaixing 总结一下上述讨论",
        root_id="root1",
        sender_username="minusjiang",
    )

    with patch("tunapi.mattermost.loop._run_engine", new_callable=AsyncMock) as run:
        await _dispatch_message(
            msg,
            cfg,
            {},
            MagicMock(),
            None,
        )

    run.assert_awaited_once()
    resolved = run.await_args.args[0]
    assert "Topic: 讨论计算器实现" in resolved.text
    assert "Participants: kaixing, codeview" in resolved.text
    assert "[kaixing]: 前端使用表单和结果区 @codeview" in resolved.text
    assert "[codeview]: 后端使用纯函数和单元测试" in resolved.text
    assert "[Current request]\n总结一下上述讨论" in resolved.text
```

- [ ] **Step 3: Run test to verify failure**

Run:

```powershell
uv run pytest tests/test_mm_loop_extra.py::test_completed_roundtable_thread_human_mention_gets_thread_context -v --no-cov
```

Expected: failure because normal `_resolve_prompt` strips the mention but does not fetch Thread history or build context.

### Task 3.2: Implement Thread context injection

**Files:**
- Modify: `src/tunapi/mattermost/loop.py`

- [ ] **Step 1: Import context helpers**

Extend the existing `from ..core.cross_roundtable import (...)` import list with:

```python
    build_thread_context_prompt,
```

- [ ] **Step 2: Add helper for prompt context loading**

Add this helper before `_resolve_prompt`:

```python
async def _prepend_thread_context(
    msg: MattermostIncomingMessage,
    cfg: MattermostBridgeConfig,
    prompt_text: str,
) -> str:
    if not msg.root_id:
        return prompt_text
    try:
        post_list = await cfg.bot._client.get_thread(msg.root_id)
    except Exception as exc:  # noqa: BLE001
        logger.warning(
            "mattermost.thread_context_load_failed",
            root_id=msg.root_id,
            error=str(exc),
        )
        return (
            "[Thread context]\n"
            "Thread history could not be loaded for this request.\n\n"
            "[Current request]\n"
            f"{prompt_text}"
        )
    if post_list is None:
        return prompt_text

    thread_posts = await _thread_posts_from_post_list(cfg, post_list)
    metadata = _find_cross_rt_metadata(thread_posts)
    state = derive_state(metadata, thread_posts) if metadata is not None else None
    return build_thread_context_prompt(
        posts=thread_posts,
        current_request=prompt_text,
        metadata=metadata,
        state=state,
        max_posts=20,
        max_chars=12_000,
    )
```

- [ ] **Step 3: Call helper from `_resolve_prompt`**

In `_resolve_prompt`, after:

```python
    prompt_text = strip_mention(prompt_text, cfg.bot_username)
    if not prompt_text:
        return None
```

add:

```python
    prompt_text = await _prepend_thread_context(msg, cfg, prompt_text)
```

The resulting block should be:

```python
    prompt_text = strip_mention(prompt_text, cfg.bot_username)
    if not prompt_text:
        return None

    prompt_text = await _prepend_thread_context(msg, cfg, prompt_text)

    return _ResolvedPrompt(text=prompt_text, file_context=file_context)
```

- [ ] **Step 4: Run focused tests**

Run:

```powershell
uv run pytest tests/test_mm_loop_extra.py::test_completed_roundtable_thread_human_mention_gets_thread_context tests/test_mattermost_cross_rt_loop.py -v --no-cov
```

Expected: tests pass.

- [ ] **Step 5: Commit context injection**

Run:

```powershell
git add src/tunapi/mattermost/loop.py tests/test_mm_loop_extra.py
@'
feat: include thread history in post-roundtable chat

Fetch Mattermost Thread history for ordinary Thread mentions and prepend compact roundtable context before dispatching to the runner.
'@ | git commit -F -
```

---

## Phase 4: Per-Bot Roundtable Busy State

### Task 4.1: Add failing tests for busy state

**Files:**
- Modify: `tests/test_mm_loop_extra.py`

- [ ] **Step 1: Add busy mention test**

Append this test:

```python
@pytest.mark.anyio()
async def test_external_mention_gets_busy_message_while_roundtable_engine_running():
    cfg = _make_cfg(bot_username="kaixing")
    cfg.cross_roundtable_enabled = True
    cfg.bot = MagicMock()
    cfg.bot.get_user = AsyncMock(return_value=MagicMock(is_bot=False))
    msg = _make_msg(
        text="@kaixing 你现在能回答另一个问题吗",
        sender_username="minusjiang",
    )

    send_calls: list[RenderedMessage] = []

    async def capture_send(message: RenderedMessage) -> None:
        send_calls.append(message)

    with patch("tunapi.mattermost.loop._is_roundtable_busy", return_value=True):
        with patch("tunapi.mattermost.loop._send_to_channel", new_callable=AsyncMock) as send:
            send.side_effect = lambda _cfg, _channel_id, message: send_calls.append(message)
            with patch("tunapi.mattermost.loop._run_engine", new_callable=AsyncMock) as run:
                await _dispatch_message(
                    msg,
                    cfg,
                    {},
                    MagicMock(),
                    None,
                )

    run.assert_not_awaited()
    assert send_calls
    assert "正在参与圆桌讨论中" in send_calls[0].text
```

- [ ] **Step 2: Add busy clear-on-error test**

Append this test:

```python
@pytest.mark.anyio()
async def test_roundtable_busy_state_clears_when_engine_raises():
    from tunapi.mattermost.loop import (
        _is_roundtable_busy,
        _run_cross_roundtable_engine,
    )

    cfg = _make_cfg(bot_username="kaixing")
    resolved = _ResolvedPrompt(text="roundtable prompt", file_context="")
    msg = _make_msg(
        text="前端视角 @kaixing",
        root_id="root1",
        sender_username="codeview",
    )

    with patch(
        "tunapi.mattermost.loop._run_engine",
        new_callable=AsyncMock,
        side_effect=RuntimeError("boom"),
    ):
        with pytest.raises(RuntimeError, match="boom"):
            await _run_cross_roundtable_engine(
                resolved,
                msg,
                cfg,
                {},
                MagicMock(),
                None,
                AsyncMock(),
            )

    assert _is_roundtable_busy(cfg) is False
```

- [ ] **Step 3: Run tests to verify failure**

Run:

```powershell
uv run pytest tests/test_mm_loop_extra.py::test_external_mention_gets_busy_message_while_roundtable_engine_running tests/test_mm_loop_extra.py::test_roundtable_busy_state_clears_when_engine_raises -v --no-cov
```

Expected: failure because busy helpers do not exist and `_run_cross_roundtable_engine` does not set busy.

### Task 4.2: Implement busy state

**Files:**
- Modify: `src/tunapi/mattermost/loop.py`

- [ ] **Step 1: Add busy state storage and helpers**

Add this near `_USER_IS_BOT_CACHE`:

```python
_ROUNDTABLE_BUSY_BOTS: set[str] = set()
```

Add these helpers after `_mentions_bot`:

```python
def _roundtable_busy_key(cfg: MattermostBridgeConfig) -> str:
    return cfg.bot_username or cfg.bot_user_id


def _is_roundtable_busy(cfg: MattermostBridgeConfig) -> bool:
    return _roundtable_busy_key(cfg) in _ROUNDTABLE_BUSY_BOTS


async def _send_busy_if_needed(
    msg: MattermostIncomingMessage,
    cfg: MattermostBridgeConfig,
    send: _SendFn,
) -> bool:
    if not _is_roundtable_busy(cfg):
        return False
    if not _mentions_bot(msg.text, cfg.bot_username):
        return False
    bot_name = cfg.bot_username.lstrip("@")
    await send(
        RenderedMessage(
            text=(
                f"@{bot_name} 正在参与圆桌讨论中，当前不接受新的外部请求。"
                "请稍后再试，或在圆桌 Thread 内继续讨论。"
            )
        )
    )
    logger.info(
        "cross_roundtable.busy_external_mention_rejected",
        bot=bot_name,
        channel_id=msg.channel_id,
        root_id=msg.root_id,
    )
    return True
```

- [ ] **Step 2: Wrap `_run_cross_roundtable_engine`**

Replace `_run_cross_roundtable_engine` body with:

```python
    key = _roundtable_busy_key(cfg)
    _ROUNDTABLE_BUSY_BOTS.add(key)
    try:
        await _run_engine(
            resolved_prompt,
            msg,
            _stateless_cross_roundtable_cfg(cfg),
            running_tasks,
            sessions,
            chat_prefs,
            send,
        )
    finally:
        _ROUNDTABLE_BUSY_BOTS.discard(key)
```

- [ ] **Step 3: Check busy before normal prompt resolution**

In `_dispatch_message`, after `_try_dispatch_command(...)` and before `_resolve_prompt(...)`, add:

```python
    if await _send_busy_if_needed(msg, cfg, send):
        return
```

The surrounding order should be:

```python
    if await _try_dispatch_command(
        msg,
        cfg,
        running_tasks,
        sessions,
        chat_prefs,
        roundtables,
        send,
        journal=journal,
        facade=facade,
        project_sessions=project_sessions,
    ):
        return

    if await _send_busy_if_needed(msg, cfg, send):
        return

    resolved = await _resolve_prompt(msg, cfg, chat_prefs, send)
```

- [ ] **Step 4: Run busy tests**

Run:

```powershell
uv run pytest tests/test_mm_loop_extra.py::test_external_mention_gets_busy_message_while_roundtable_engine_running tests/test_mm_loop_extra.py::test_roundtable_busy_state_clears_when_engine_raises -v --no-cov
```

Expected: busy tests pass.

- [ ] **Step 5: Run all focused continuation tests**

Run:

```powershell
uv run pytest tests/test_cross_roundtable.py tests/test_mattermost_cross_rt_loop.py tests/test_mm_loop_extra.py -v --no-cov
```

Expected: tests pass, or unrelated pre-existing failures are recorded with exact failing test names.

- [ ] **Step 6: Commit busy state**

Run:

```powershell
git add src/tunapi/mattermost/loop.py tests/test_mm_loop_extra.py
@'
feat: reject external mentions while roundtable bot is busy

Track per-process roundtable engine activity and send a busy message instead of starting overlapping normal chat runs.
'@ | git commit -F -
```

---

## Phase 5: End-To-End Verification

### Task 5.1: Run focused suite

**Files:** none

- [ ] **Step 1: Run roundtable and Mattermost loop tests**

Run:

```powershell
uv run pytest tests/test_cross_roundtable.py tests/test_cross_rt_commands.py tests/test_mattermost_cross_rt_loop.py tests/test_mm_loop_extra.py -v --no-cov
```

Expected: tests pass, or unrelated pre-existing failures are recorded with exact test names and traceback summaries.

### Task 5.2: Run broader Mattermost suite

**Files:** none

- [ ] **Step 1: Run broader transport tests**

Run:

```powershell
uv run pytest tests/test_mattermost_roundtable.py tests/test_mm_commands_extra.py tests/test_archive_roundtable_wiring.py tests/test_mattermost_cross_rt_loop.py tests/test_mm_loop_extra.py -v --no-cov
```

Expected: Mattermost transport tests pass, or environment-related failures are recorded separately.

### Task 5.3: Manual CVM acceptance after deploy

**Files:** none

- [ ] **Step 1: Deploy the tunaPi feature branch to CVM**

Use the existing deployment process that installs the `feat/multi-agent-roundtable` branch for both `kaixing` and `codeview`.

Expected: both systemd user services are active.

- [ ] **Step 2: Start a roundtable**

In Mattermost target channel:

```text
!rt start @kaixing @codeview 讨论一下我如果要实现一个计算器，应该怎么做？
```

Expected: bots discuss in the Thread and stop at `max_rounds` or when a bot omits the next mention.

- [ ] **Step 3: Continue in the completed Thread**

In the same Thread:

```text
@kaixing 总结一下上述讨论之后的实现方案
```

Expected: `kaixing` replies in the same Thread and references the previous `kaixing` and `codeview` discussion.

- [ ] **Step 4: Verify no bot loop**

Wait 60 seconds after the follow-up reply.

Expected: no automatic bot-to-bot conversation restarts unless the user explicitly sends `!rt resume` in a paused Thread.

---

## Self-Review Checklist

- [ ] Active roundtable bot-to-bot routing remains strict and ordered.
- [ ] Non-active human `@bot` in the same Thread falls through to normal chat.
- [ ] Non-active bot-authored messages are ignored and cannot restart loops.
- [ ] Thread context prompt includes topic, participants, status, prior bot discussion, and current request.
- [ ] Busy state is per process and clears in a `finally` block.
- [ ] No Mattermost server changes are required.
- [ ] Tests cover core formatting, routing release, context injection, busy rejection, and busy cleanup.
