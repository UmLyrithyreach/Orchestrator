# orchestrator

An agent skill that fans a task out to parallel subagents, and lets **you** choose the roles and the models. Works in [Hermes Agent](https://hermes-agent.nousresearch.com), [Claude Code](https://claude.com/claude-code), [OpenCode](https://opencode.ai) and Rayu.

Most delegation skills hardcode a subagent model. Yours won't match mine. This one asks.

## What it does

1. Scopes the task. Tells you if one agent is enough.
2. Asks which roles to run: implement, review, test, explore, or your own.
3. Asks which model the subagents should use (configured, current, pasted id, or per-role).
4. Dispatches all packets in parallel.
5. Checks every result against its acceptance criteria. Rejects and redoes up to 3 times. Reports.

The skill detects which agent it is running in and uses that agent's own tools:

| Agent | Asks with | Spawns with | Subagent model set by |
|---|---|---|---|
| Hermes Agent | `clarify` | `delegate_task` | `delegation.model` config |
| Claude Code, Rayu | `AskUserQuestion` | `Agent` | `model` on each call |
| OpenCode | `question` | `task` | the subagent's agent definition |

## Install

```bash
git clone https://github.com/UmLyrithyreach/Orchestrator.git
```

Then copy the `orchestrator` folder into your agent's skills directory:

| Agent | Command |
|---|---|
| Hermes Agent | `cp -r Orchestrator/orchestrator ~/.hermes/skills/` |
| Claude Code | `cp -r Orchestrator/orchestrator ~/.claude/skills/` |
| OpenCode | `cp -r Orchestrator/orchestrator ~/.config/opencode/skills/` |
| Rayu | `cp -r Orchestrator/orchestrator ~/.rayu/skills/` |

OpenCode also reads `~/.claude/skills/`, so one copy there covers both.

Start a new session and say: `orchestrate: add dark mode to the settings page`.

## Example

```
> orchestrate: add rate limiting to the API and cover it with tests

Scope: 2 independent parts (code, tests). Suggest: explore, implement, test.
? Roles  [x] explore  [x] implement  [ ] review  [x] test
? Model  > Use configured: my-fast-model
Dispatching 1 explore packet... done. Dispatching implement + test...
implement: GO (diff read). test: 14/14 pass. Shipped.
```

## Requirements

An agent that can ask questions and spawn subagents. Per agent:

- **Hermes Agent**: `delegate_task` and `clarify`. Optional: `hermes config set delegation.model <id>` to pin the subagent model.
- **Claude Code / Rayu**: nothing extra.
- **OpenCode**: to run subagents on a specific model, define a subagent with that `model` (see `orchestrator/references/opencode.md`). The question tool is on in the TUI; for `opencode run`, set `OPENCODE_ENABLE_QUESTION_TOOL=1`.
