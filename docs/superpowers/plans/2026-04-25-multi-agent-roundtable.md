# Multi-Agent Roundtable Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a cross-instance Mattermost roundtable where two independent tunaPi bots discuss inside one Mattermost Thread, while humans can observe, interrupt, pause, resume, and close the discussion.

**Architecture:** Mattermost Thread history is the shared source of truth. tunaPi implements command parsing, Thread metadata, state derivation, prompt construction, and Thread routing. The `mattermost` repo owns deployment automation. `codex` stays an external runner dependency unless integration proves it must change.

**Tech Stack:** Python 3.14, tunaPi, Mattermost WebSocket/API, Mattermost Thread replies, PowerShell deployment entrypoint, Linux shell deployment script, systemd user services.

---

## Decisions Locked In

- User-facing name is **multi-agent roundtable**.
- Internal code name can be `cross_roundtable` because the feature crosses tunaPi process boundaries.
- Do not use a per-instance local JSON file as the authoritative cross-instance state store.
- Mattermost Thread history is authoritative for metadata, control events, participant messages, human replies, round count, and recovery.
- Local tunaPi state is allowed only as a cache.
- Main code changes happen in `tunaPi`.
- One-click deployment lives in the outer `mattermost` workspace/repo.
- `codex` is not modified for the first implementation pass.

---

## Repository Plan

| Repo | Branch | What changes here |
|------|--------|-------------------|
| `tunaPi` | `feat/multi-agent-roundtable` | Feature implementation and tests |
| `mattermost` | `feat/multi-agent-roundtable-deploy` | Deployment scripts, config templates, service templates, verification scripts |
| `codex` | none | No changes in phase 1; document required version/PATH only |

Commit/push rule: commit in the repository that owns the changed files. Do not commit tunaPi code from the outer `mattermost` repository if `tunaPi` is an independent Git repo.

---

## File Structure

### tunaPi

| File | Action | Responsibility |
|------|--------|----------------|
| `src/tunapi/core/cross_roundtable.py` | Create | Pure state derivation, metadata model, transcript parsing, next-speaker logic, prompt builder |
| `src/tunapi/mattermost/commands.py` | Modify | Parse `!rt start/stop/resume/close/status` while preserving existing `!rt "topic"` |
| `src/tunapi/mattermost/types.py` | Modify | Add immutable fields only if needed, such as `from_bot` |
| `src/tunapi/mattermost/parsing.py` | Modify | Populate any new fields during dataclass construction, not by mutation |
| `src/tunapi/mattermost/loop.py` | Modify | Route roundtable Thread messages and control commands |
| `src/tunapi/mattermost/client_api.py` | Modify | Expose `get_thread(root_post_id)` using Mattermost `/posts/{root_post_id}/thread` |
| `src/tunapi/settings.py` | Modify | Add `[transports.mattermost.cross_roundtable]` config model |
| `src/tunapi/mattermost/backend.py` | Modify | Pass cross-roundtable settings into bridge config |
| `src/tunapi/mattermost/bridge.py` | Modify | Store cross-roundtable settings on `MattermostBridgeConfig` |
| `tests/test_cross_roundtable.py` | Create | Pure unit tests |
| `tests/test_cross_rt_commands.py` | Create | Command parser tests |
| `tests/test_mattermost_cross_rt_loop.py` | Create | Mattermost routing tests with fakes |

### mattermost

| File | Action | Responsibility |
|------|--------|----------------|
| `deploy/roundtable/deploy.ps1` | Create | Windows local one-click entrypoint |
| `deploy/roundtable/deploy.sh` | Create | Remote Linux deployment implementation |
| `deploy/roundtable/verify.sh` | Create | Remote health checks |
| `deploy/roundtable/templates/tunapi-kaixing.toml.template` | Create | First tunaPi config template |
| `deploy/roundtable/templates/tunapi-agent2.toml.template` | Create | Second tunaPi config template |
| `deploy/roundtable/templates/tunapi-kaixing.service.template` | Create | systemd user service for kaixing |
| `deploy/roundtable/templates/tunapi-agent2.service.template` | Create | systemd user service for agent2 |
| `deploy/roundtable/README.md` | Create | Deployment usage and rollback notes |

---

## Phase 0: Branch And Baseline

### Task 0.1: Create feature branches

**Files:** none

- [ ] In `tunaPi`, create `feat/multi-agent-roundtable`.

Run:

```bash
cd /e/Github/mattermost/tunaPi
git checkout -b feat/multi-agent-roundtable
```

Expected: branch created.

- [ ] In outer `mattermost`, create `feat/multi-agent-roundtable-deploy`.

Run:

```bash
cd /e/Github/mattermost
git checkout -b feat/multi-agent-roundtable-deploy
```

Expected: branch created.

### Task 0.2: Verify current tunaPi baseline

**Files:** none

- [ ] Run focused existing tests before editing.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_mattermost_roundtable.py tests/test_mm_commands_extra.py -v --no-cov
```

Expected: tests pass, or failures are recorded as pre-existing before implementation begins.

---

## Phase 1: Pure tunaPi Roundtable Core

### Task 1.1: Add metadata and transcript parsing tests

**Files:**
- Create: `tunaPi/tests/test_cross_roundtable.py`

- [ ] Write tests for parsing Thread metadata from a root/system message.

Test intent:

```python
from tunapi.core.cross_roundtable import parse_metadata


def test_parse_metadata_from_system_block():
    text = """🎯 圆桌会议已开启

<!-- tunapi:roundtable {"version":1,"topic":"分析架构","participants":["kaixing","agent2"],"max_rounds":3} -->
"""

    metadata = parse_metadata(text)

    assert metadata.topic == "分析架构"
    assert metadata.participants == ["kaixing", "agent2"]
    assert metadata.max_rounds == 3
```

- [ ] Run and confirm failure.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py -v
```

Expected: import or function missing failure.

### Task 1.2: Implement metadata model and parser

**Files:**
- Create: `tunaPi/src/tunapi/core/cross_roundtable.py`

- [ ] Implement `CrossRTMetadata`, `parse_metadata()`, and `format_metadata_marker()`.

Implementation constraints:

- Use a hidden HTML comment marker so humans see a clean Thread while tunaPi can parse structured state.
- Use JSON inside the marker.
- Do not parse human-readable Chinese labels as the source of truth.

- [ ] Run tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py -v
```

Expected: metadata tests pass.

### Task 1.3: Add state derivation tests

**Files:**
- Modify: `tunaPi/tests/test_cross_roundtable.py`

- [ ] Test state is derived from Thread posts, not local store.

Test cases:

- No control event after start means `active`.
- Last control event `pause` means `paused`.
- Last control event `resume` after pause means `active`.
- Control event `close` means `closed`.
- Enough participant replies means `closed` by max rounds.
- Human replies appear in prompt context but do not increment round count.

- [ ] Run and confirm failure.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py -v
```

Expected: state derivation functions missing.

### Task 1.4: Implement state derivation

**Files:**
- Modify: `tunaPi/src/tunapi/core/cross_roundtable.py`

- [ ] Implement:

```python
class CrossRTStatus(str, Enum):
    ACTIVE = "active"
    PAUSED = "paused"
    CLOSED = "closed"


def derive_state(metadata: CrossRTMetadata, posts: list[ThreadPost]) -> CrossRTState:
    control = latest_control_event(posts)
    completed_rounds = count_completed_rounds(posts, metadata.participants)
    if control == "close" or completed_rounds >= metadata.max_rounds:
        status = CrossRTStatus.CLOSED
    elif control == "pause":
        status = CrossRTStatus.PAUSED
    else:
        status = CrossRTStatus.ACTIVE
    return CrossRTState(
        metadata=metadata,
        status=status,
        current_round=completed_rounds,
        next_participant=resolve_next_participant(posts, metadata.participants),
    )
```

Implementation rules:

- `ThreadPost` should be a small transport-neutral dataclass with `sender_username`, `message`, `created_at`, and `root_id`.
- Only messages from `metadata.participants` count as agent turns.
- A round is complete when each participant has spoken once since the previous round boundary.
- Control events are structured markers, not natural language.
- Closed beats paused; paused beats active.

- [ ] Run tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py -v
```

Expected: state derivation tests pass.

### Task 1.5: Add prompt builder tests

**Files:**
- Modify: `tunaPi/tests/test_cross_roundtable.py`

- [ ] Test prompt includes topic, participant list, previous bot turns, human comments, and next participant instruction.

- [ ] Implement `build_agent_prompt()`.

**Files:**
- Modify: `tunaPi/src/tunapi/core/cross_roundtable.py`

Expected behavior:

- Prompt instructs the current bot to answer concisely.
- Prompt tells the current bot who to `@mention` next if the discussion should continue.
- Prompt tells the current bot not to `@mention` anyone if the discussion has converged.
- Prompt includes human Thread replies as discussion context.

- [ ] Run tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py -v
```

Expected: all pure core tests pass.

- [ ] Commit tunaPi core work.

Run:

```bash
cd /e/Github/mattermost/tunaPi
git add src/tunapi/core/cross_roundtable.py tests/test_cross_roundtable.py
git commit -m "feat: add multi-agent roundtable state core"
```

---

## Phase 2: tunaPi Command And Settings

### Task 2.1: Add command parser tests

**Files:**
- Create: `tunaPi/tests/test_cross_rt_commands.py`

- [ ] Test `!rt start @kaixing @agent2 topic` parsing.

Required cases:

- Two participants and Chinese topic.
- More than two participants accepted by parser, even if deployment initially uses two.
- One participant returns a clear error.
- No topic returns a clear error.
- Existing `!rt "topic"` behavior remains unchanged.

- [ ] Run and confirm failure.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_rt_commands.py -v
```

Expected: parser missing failure.

### Task 2.2: Implement command parsing

**Files:**
- Modify: `tunaPi/src/tunapi/mattermost/commands.py`

- [ ] Implement `parse_cross_rt_start(args: str)`.

Constraints:

- Only mentions before the topic count as participants.
- Strip leading `@`.
- Preserve Unicode topic text.
- Return `(participants, topic, error)` to match existing command handler style.

- [ ] Run tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_rt_commands.py tests/test_mm_commands_extra.py -v --no-cov
```

Expected: command tests pass and existing command tests still pass.

### Task 2.3: Add settings and bridge plumbing

**Files:**
- Modify: `tunaPi/src/tunapi/settings.py`
- Modify: `tunaPi/src/tunapi/mattermost/backend.py`
- Modify: `tunaPi/src/tunapi/mattermost/bridge.py`

- [ ] Add settings model:

```toml
[transports.mattermost.cross_roundtable]
enabled = true
max_rounds = 3
timeout_minutes = 5
```

- [ ] Pass these values into `MattermostBridgeConfig`.

- [ ] Add or update settings tests for loading `[transports.mattermost.cross_roundtable]`.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_settings.py tests/test_mm_commands_extra.py -v --no-cov
```

Expected: settings tests and existing command tests pass.

- [ ] Commit command and settings work.

Run:

```bash
cd /e/Github/mattermost/tunaPi
git add src/tunapi/mattermost/commands.py src/tunapi/settings.py src/tunapi/mattermost/backend.py src/tunapi/mattermost/bridge.py tests/test_cross_rt_commands.py
git commit -m "feat: add multi-agent roundtable commands and settings"
```

---

## Phase 3: Mattermost Thread API And Routing

### Task 3.1: Add immutable parsing support if needed

**Files:**
- Modify: `tunaPi/src/tunapi/mattermost/types.py`
- Modify: `tunaPi/src/tunapi/mattermost/parsing.py`

- [ ] If a `from_bot` field is needed, add it to the frozen dataclass and pass it during construction.

Important: do not mutate `MattermostIncomingMessage` after construction because it is `frozen=True`.

- [ ] Prefer `sender_username in metadata.participants` for participant detection. Use `from_bot` only as a safety signal, not as the sole source of truth.

- [ ] Run parsing tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_mattermost_parsing.py -v --no-cov
```

Expected: parsing tests pass.

### Task 3.2: Expose Thread fetch API

**Files:**
- Modify: `tunaPi/src/tunapi/mattermost/client_api.py`

- [ ] Add or reuse a method equivalent to:

```python
from .api_models import PostList


async def get_thread(self, root_post_id: str) -> PostList | None:
    result = await self._get(f"/posts/{root_post_id}/thread")
    return self._decode_result(method="get_thread", payload=result, model=PostList)
```

Expected endpoint:

```text
GET /api/v4/posts/{root_post_id}/thread
```

- [ ] Convert Mattermost posts into the core `ThreadPost` shape before state derivation.

### Task 3.3: Add routing tests

**Files:**
- Create: `tunaPi/tests/test_mattermost_cross_rt_loop.py`

- [ ] Test `!rt start` sends a root post and a Thread kickoff message containing metadata marker.
- [ ] Test a Thread mention to current bot fetches Thread history and calls the runner with a roundtable prompt.
- [ ] Test paused Thread does not call runner.
- [ ] Test closed Thread does not call runner.
- [ ] Test human Thread reply is ignored immediately but included in next prompt.
- [ ] Test existing non-start `!rt "topic"` still routes to old single-instance roundtable.

Use fake Mattermost transport/client objects rather than hitting a real server.

### Task 3.4: Implement loop integration

**Files:**
- Modify: `tunaPi/src/tunapi/mattermost/loop.py`

- [ ] Add `_start_multi_agent_roundtable()`.

Expected behavior:

- Send root post in channel.
- Include human-readable metadata.
- Include hidden structured metadata marker.
- Send kickoff message in Thread mentioning the first participant.

- [ ] Add `_try_dispatch_cross_roundtable()`.

Expected behavior:

- Only inspect messages with `root_id`.
- Fetch Thread history.
- Parse metadata.
- Derive state.
- Handle `!rt stop/resume/close/status` as structured control events.
- If the message mentions the current bot and state is active, build prompt and run engine.
- If state is paused or closed, do not run engine.

- [ ] Route `!rt start` before falling through to existing `handle_rt()`.

- [ ] Run focused tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py tests/test_cross_rt_commands.py tests/test_mattermost_cross_rt_loop.py -v --no-cov
```

Expected: all focused tests pass.

- [ ] Run broader Mattermost tests.

Run:

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_mattermost_roundtable.py tests/test_mm_commands_extra.py tests/test_archive_roundtable_wiring.py -v --no-cov
```

Expected: existing Mattermost behavior still passes.

- [ ] Commit Mattermost integration.

Run:

```bash
cd /e/Github/mattermost/tunaPi
git add src/tunapi/mattermost src/tunapi/core/cross_roundtable.py tests/test_mattermost_cross_rt_loop.py
git commit -m "feat: route multi-agent roundtables through Mattermost threads"
```

---

## Phase 4: tunaPi End-To-End Local Verification

### Task 4.1: Run tunaPi test suite

**Files:** none

- [ ] Run focused suite first.

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/test_cross_roundtable.py tests/test_cross_rt_commands.py tests/test_mattermost_cross_rt_loop.py -v --no-cov
```

Expected: pass.

- [ ] Run all tests if focused suite is green.

```bash
cd /e/Github/mattermost/tunaPi
uv run pytest tests/ -v --no-cov -x
```

Expected: pass or stop on first unrelated pre-existing failure.

### Task 4.2: Push tunaPi branch

**Files:** none

- [ ] Push feature branch.

```bash
cd /e/Github/mattermost/tunaPi
git push -u origin feat/multi-agent-roundtable
```

Expected: remote branch exists and can be installed on CVM.

---

## Phase 5: One-Click Deployment In mattermost Repo

### Task 5.1: Add deployment templates

**Files:**
- Create: `deploy/roundtable/templates/tunapi-kaixing.toml.template`
- Create: `deploy/roundtable/templates/tunapi-agent2.toml.template`
- Create: `deploy/roundtable/templates/tunapi-kaixing.service.template`
- Create: `deploy/roundtable/templates/tunapi-agent2.service.template`

- [ ] Add config template fields:

```text
{{MATTERMOST_URL}}
{{KAIXING_TOKEN}}
{{AGENT2_TOKEN}}
{{TUNAPI_BRANCH}}
{{CODEX_PATH}}
```

- [ ] Both configs include:

```toml
[transports.mattermost]
session_mode = "chat"
trigger_mode = "mentions"

[transports.mattermost.cross_roundtable]
enabled = true
max_rounds = 3
timeout_minutes = 5
```

- [ ] systemd services set distinct `TUNAPI_CONFIG_DIR` values:

```text
/data/home/minusjiang/.tunapi
/data/home/minusjiang/.tunapi-agent2
```

### Task 5.2: Add remote deploy script

**Files:**
- Create: `deploy/roundtable/deploy.sh`

- [ ] Script accepts:

```bash
./deploy.sh --tunapi-branch feat/multi-agent-roundtable --mattermost-url http://9.134.128.138:8065
```

- [ ] Script performs:

1. Verify `python3`, `uv`, `git`, `systemctl --user`, and codex binary.
2. Clone or update tunaPi on the remote host.
3. Checkout requested tunaPi branch.
4. Install tunaPi from that checkout.
5. Render two tunaPi config files from templates.
6. Install two systemd user services.
7. Run `systemctl --user daemon-reload`.
8. Restart `tunapi-kaixing.service` and `tunapi-agent2.service`.
9. Run `verify.sh`.

Do not echo bot tokens.

### Task 5.3: Add Windows one-click entrypoint

**Files:**
- Create: `deploy/roundtable/deploy.ps1`

- [ ] Script accepts:

```powershell
.\deploy\roundtable\deploy.ps1 `
  -Host 9.134.128.138 `
  -TunapiBranch feat/multi-agent-roundtable `
  -MattermostUrl http://9.134.128.138:8065
```

- [ ] Script uploads deployment files with `scp`.
- [ ] Script runs remote `deploy.sh` over `ssh`.
- [ ] Script reads tokens from local environment variables:

```powershell
$env:KAIXING_BOT_TOKEN
$env:AGENT2_BOT_TOKEN
```

Do not write tokens into Git-tracked files.

### Task 5.4: Add verification script

**Files:**
- Create: `deploy/roundtable/verify.sh`

- [ ] Verify:

1. Two tunaPi systemd user services are active.
2. Two heartbeat files are fresh.
3. Logs have no recent traceback.
4. Both config directories exist.
5. `codex` is found on PATH used by services.

- [ ] Print a concise pass/fail summary.

### Task 5.5: Add deployment README

**Files:**
- Create: `deploy/roundtable/README.md`

- [ ] Document prerequisites:

- Mattermost bot `kaixing` exists and has token.
- Mattermost bot `agent2` exists and has token.
- Both bots are invited to target channel.
- Remote host has SSH access.
- Remote host has codex installed.

- [ ] Document deploy command, verify command, rollback command, and log locations.

### Task 5.6: Commit deployment automation

Run:

```bash
cd /e/Github/mattermost
git add deploy/roundtable
git commit -m "feat: add multi-agent roundtable deployment automation"
git push -u origin feat/multi-agent-roundtable-deploy
```

---

## Phase 6: Remote Deployment And E2E

### Task 6.1: Prepare Mattermost bot account

Manual actions:

- [ ] Create or verify `@agent2` bot account in Mattermost.
- [ ] Copy token into local environment variable, not into a file.
- [ ] Invite `@agent2` to the target channel.

### Task 6.2: Deploy to CVM

Run from local Windows workspace:

```powershell
cd e:\Github\mattermost
$env:KAIXING_BOT_TOKEN = "<existing-token>"
$env:AGENT2_BOT_TOKEN = "<new-token>"
.\deploy\roundtable\deploy.ps1 -Host 9.134.128.138 -TunapiBranch feat/multi-agent-roundtable -MattermostUrl http://9.134.128.138:8065
```

Expected:

- `tunapi-kaixing.service` active.
- `tunapi-agent2.service` active.
- Verification script passes.

### Task 6.3: Smoke test each bot

In Mattermost:

```text
@kaixing hello
@agent2 hello
```

Expected: each bot responds only when mentioned.

### Task 6.4: End-to-end roundtable test

In target Mattermost channel:

```text
!rt start @kaixing @agent2 分析 codebuddy-mem 的架构设计
```

Expected:

1. Root post appears in channel.
2. Thread contains kickoff message.
3. `@kaixing` responds in Thread.
4. `@agent2` responds in Thread.
5. Human Thread reply is included in next prompt.
6. Roundtable stops when max rounds is reached or when a bot omits the next mention.

### Task 6.5: Control command tests

In the active Thread:

```text
!rt stop
```

Expected: Thread records paused control event and bots stop.

```text
!rt resume
```

Expected: Thread records resumed control event and next mention can continue.

```text
!rt close
```

Expected: Thread records closed control event and bots stop permanently for that Thread.

---

## Rollback

Remote rollback should be service-level first:

```bash
systemctl --user stop tunapi-agent2.service
systemctl --user restart tunapi-kaixing.service
```

If the new tunaPi branch is bad:

```bash
cd /data/home/minusjiang/src/tunaPi
git checkout main
uv tool install --force .
systemctl --user restart tunapi-kaixing.service tunapi-agent2.service
```

If `agent2` causes noise, remove or disable the bot token and leave `kaixing` running.

---

## Self-Review Checklist

- [ ] No task depends on per-instance JSON as authoritative cross-instance state.
- [ ] Existing single-instance `!rt "topic"` remains supported.
- [ ] `MattermostIncomingMessage` immutability is respected.
- [ ] `codex` is not modified in phase 1.
- [ ] Deployment scripts do not print or commit tokens.
- [ ] One-click deployment restarts two named services, not anonymous `nohup` processes.
- [ ] E2E tests cover start, pause, resume, close, human reply, and max rounds.
