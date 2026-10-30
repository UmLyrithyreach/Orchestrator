# orchestrator

An agent skill that turns your main model into the manager. It splits the task, fans it out to parallel subagents with the roles and models **you** choose, checks what comes back, and approves it or sends it back to be redone. It never touches the code itself. Works in [Hermes Agent](https://hermes-agent.nousresearch.com), [Claude Code](https://claude.com/claude-code), [OpenCode](https://opencode.ai), Rayu and [Antigravity CLI](https://antigravity.google).

Most delegation skills hardcode a subagent model. Yours won't match mine. This one asks.

## Why

- **Your main model stays lean.** Subagents read files, write code and run tests in their own contexts. The main model only sees the task packets, the diffs and the reports, so its context and its token bill stay small on big tasks.
- **Cheap work goes to cheap models.** Pick a fast model for explore and test, a strong one for review, and keep your expensive model for managing.
- **Nothing ships unchecked.** Every result is checked against its acceptance criteria before it counts as done.

Fanning out isn't free: each subagent pays for its own context, so for a one-file edit a single agent is cheaper. The skill tells you when that's the case.

## What it does

1. Scopes the task. Tells you if one agent is enough.
2. Asks which roles to run: implement, review, test, explore, or your own.
3. Asks which model the subagents should use (configured, current, pasted id, or per-role).
4. Splits each role into independent units and fans every role out: three implement units become three implement subagents, run in parallel on the role's model. Waves go explore → implement → review/test.
5. Checks every result (the diff and the test evidence) against its acceptance criteria. GO approves it; REJECT sends a corrected packet back, up to 3 times. Reports.

The skill detects which agent it is running in and uses that agent's own tools:

| Agent | Asks with | Spawns with | Subagent model set by |
|---|---|---|---|
| Hermes Agent | `clarify` | `delegate_task` | `delegation.model` config |
| Claude Code, Rayu | `AskUserQuestion` | `Agent` | `model` on each call |
| OpenCode | `question` | `task` | the subagent's agent definition |
| Antigravity CLI | `ask_question` | `invoke_subagent` | the session's model (`agy --model`) |

## Install

Easiest: paste this into your agent.

```
Install the skill from https://github.com/UmLyrithyreach/Orchestrator
```

It clones the repo and copies the `orchestrator` folder into its own skills directory. To do it by hand:

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
| Antigravity CLI | `mkdir -p ~/.gemini/config/skills && cp -r Orchestrator/orchestrator ~/.gemini/config/skills/` |

OpenCode also reads `~/.claude/skills/`, so one copy there covers both. In Antigravity CLI, run `/skills reload` or start a new session.

Start a new session and say: `orchestrate: add dark mode to the settings page`.

## Example

```
> orchestrate: add rate limiting to the API and cover it with tests

Scope: 2 independent parts (code, tests). Suggest: explore, implement, test.
? Roles  [x] explore  [x] implement  [ ] review  [x] test
? Model  > Use configured: my-fast-model
Plan: explore ×1 · implement ×3 (middleware, config, routes) · test ×2 (unit, integration)
Wave 1: explore ×1... GO.
Wave 2: implement ×3 in parallel... middleware GO, config GO, routes REJECT -> redo -> GO.
Wave 3: test ×2 in parallel... unit 9/9, integration 5/5. Shipped.
```

## Requirements

An agent that can ask questions and spawn subagents. Per agent:

- **Hermes Agent**: `delegate_task` and `clarify`. Optional: `hermes config set delegation.model <id>` to pin the subagent model.
- **Claude Code / Rayu**: nothing extra.
- **OpenCode**: to run subagents on a specific model, define a subagent with that `model` (see `orchestrator/references/opencode.md`). The question tool is on in the TUI; for `opencode run`, set `OPENCODE_ENABLE_QUESTION_TOOL=1`.
- **Antigravity CLI**: subagents can't take a model per call; start the session on the model you want them to use (`agy --model <id>`).
