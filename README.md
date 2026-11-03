# orchestrator

An agent skill that turns your main model into the manager. It splits the task, fans it out to parallel subagents with the roles and models **you** choose, checks what comes back, and approves it or sends it back to be redone. It never touches the code itself. Works in [Hermes Agent](https://hermes-agent.nousresearch.com), [Claude Code](https://claude.com/claude-code), [OpenCode](https://opencode.ai), Rayu and [Antigravity CLI](https://antigravity.google).

Most delegation skills hardcode a subagent model. Yours won't match mine. This one asks.

## Why

- **Your main model stays lean.** Subagents read files, write code and run tests in their own contexts. The main model only sees the task packets, the diffs and the reports, so its context and its token bill stay small on big tasks.
- **Cheap work goes to cheap models.** Pick a fast model for explore and test, a strong one for review, and keep your expensive model for managing.
- **Nothing ships unchecked.** Every packet ends with gates: a command and the text its output must contain. The manager re-runs them itself before anything counts as done.

Fanning out isn't free: each subagent pays for its own context, so for a one-file edit a single agent is cheaper. The skill tells you when that's the case.

## What it does

1. Scopes the task. Tells you if one agent is enough.
2. Asks which roles to run: implement, review, test, explore, or your own.
3. Asks which model the subagents should use (configured, current, pasted id, or per-role).
4. Splits each role into independent units and fans every role out: three implement units become three implement subagents, run in parallel on the role's model. Waves go explore → implement → review/test.
5. Checks every result: re-runs each packet's gates itself (a fix must also fail its test with the fix set aside), reads the diff, and gives review packets only the spec and the diff. GO merges the packet's branch. The first REJECT sends a corrected packet to a fresh subagent; the second redoes it on a stronger model; after that it stops and shows you the evidence. Reports.

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

Scope: 2 independent parts (code, tests). Suggest: explore, implement, review, test.
? Roles  [x] explore  [x] implement  [x] review  [x] test
? Model  > Per-role: explore, test my-fast-model · implement my-mid-model · review my-strong-model
Plan: explore ×1 · implement ×3 (middleware, config, routes) · review ×3 · test ×1 (full suite)
Wave 1: explore ×1... GO.
Wave 2: implement ×3 in parallel, one worktree each.
  middleware  reports G1-G2 pass. Re-run: pass. Review (spec + diff): no findings. GO, merged.
  config      reports G1 pass. Re-run: pass. GO, merged.
  routes      reports G1-G3 pass. Re-run: G3 FAIL (no 429 over the limit). REJECT.
              Corrected packet, fresh subagent, same model. Re-run: G3 FAIL. REJECT.
              Redo in a fresh subagent on my-strong-model. Re-run: pass. GO, merged.
Wave 3: test ×1, full suite after merge... reports pass. Re-run: pass. Shipped.
```

## Don't combine with

Other plugins or skills that are always on and set conflicting delegation rules, e.g. one that lets the orchestrator edit code, or one that never asks before spawning. Use one at a time.

## Requirements

An agent that can ask questions and spawn subagents. Per agent:

- **Hermes Agent**: `delegate_task` and `clarify`. Optional: `hermes config set delegation.model <id>` to pin the subagent model.
- **Claude Code / Rayu**: nothing extra.
- **OpenCode**: to run subagents on a specific model, define a subagent with that `model` (see `orchestrator/references/opencode.md`). The question tool is on in the TUI; for `opencode run`, set `OPENCODE_ENABLE_QUESTION_TOOL=1`.
- **Antigravity CLI**: subagents can't take a model per call; start the session on the model you want them to use (`agy --model <id>`).
