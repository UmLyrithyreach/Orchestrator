---
name: orchestrator
description: Use when the user says "orchestrate", "fan out", "spawn agents", "split this across subagents", or gives a task with 2+ independent workstreams (research + implement + test). Asks which roles and which model to use, dispatches parallel subagents, then verifies results before reporting.
version: 1.1.0
license: MIT
compatibility: Hermes Agent, Claude Code, OpenCode, Rayu, Antigravity CLI
metadata:
  hermes:
    tags: [delegation, subagents, orchestration, parallel]
---

# Orchestrator

You stay the orchestrator. You plan, dispatch, verify, and report. Subagents do the heavy work.

Works in any agent that can spawn subagents. Your model stays whatever the user picked for this session; model choices below apply only to subagents.

## Your tools

Find your harness by the tools you have, then read its reference once before the first dispatch.

| Harness | Ask the user | Spawn a subagent | Reference |
|---|---|---|---|
| Hermes Agent | `clarify` (`multi_select`) | `delegate_task` (`tasks` array) | `references/hermes.md` |
| Claude Code, Rayu | `AskUserQuestion` (`multiSelect`) | `Agent` / `Task` | `references/claude-code.md` |
| OpenCode | `question` (`multiple`) | `task` | `references/opencode.md` |
| Antigravity CLI (`agy`) | `ask_question` | `invoke_subagent` (`subagents` list) | `references/antigravity.md` |

Below, **ask** means your ask tool and **spawn** means your spawn tool. With no ask tool, ask in plain text as a numbered list and wait for the answer. With no spawn tool, say so and do the work yourself, in one agent.

## When NOT to use

If the task is one small edit, one lookup, or one file, do it yourself and say so. Fan-out costs tokens and adds coordination overhead. Recommend the minimal set (often 1 agent, or none) before offering the full menu.

## Hard rules

1. **Ask before spawning.** Show the role menu and model picker. Skip a question only if the user already named that choice in their request, or said "same as last time" (see Session memory).
2. **Fan out in one step.** All independent packets go out together so they run in parallel: one `delegate_task` or `invoke_subagent` call, or several spawn calls in one message.
3. **Disjoint files.** Each packet owns its own files. If two roles need one file, one owns it; others get a handoff note.
4. **Delegate the work, keep the glue.** Orchestrator dispatches, reads results, reports. Implementation goes to implement packets. Plan files, todo bookkeeping, and user questions are yours.
5. **Self-contained packets.** Subagents cannot see this conversation. See the packet template.
6. **Verify before GO.** Subagent reports are self-reports. Read the diff or run the check yourself before accepting.

## Workflow

### 1. Scope
Restate the task in one sentence. Sketch the decomposition. If it fits one agent, say so and offer that.

### 2. Role menu
**Ask**, multi-select. Mark the suggested roles.

| Role | Does |
|---|---|
| implement | Writes the changes. Split into several packets if work partitions cleanly. |
| review | Critiques plan or output. Findings only, no edits. |
| test | Runs builds/tests/linters. Reports pass/fail with evidence. |
| explore | Recon first, so implement packets get exact paths and line numbers. |

The user can type a custom role via "Other" (e.g. docs, security-audit, migrate). Treat it as another packet with its own Owns/Accept criteria.

Sequencing: `explore` before `implement` on unknown ground. `review` and `test` after `implement`, in a second dispatch.

### 3. Model picker
Never assume a model. Every user has different providers.

Each harness sets the subagent model differently; your reference says how. Offer, in this order:

1. **Use configured / default**: the subagent model already set up (name it if you can read it).
2. **Match my current model**: the session's model, passed explicitly.
3. **Paste a model id**: the user types the exact id; pass it unchanged. If it looks like an alias or partial name, resolve it and confirm once.
4. Optional: **Per-role models** (e.g. cheap model for explore, strong model for review). Ask once per role.

If the harness cannot apply the pick, say so plainly and offer what it can do. Never claim a model you did not set.

### 4. Dispatch
**Spawn** one subagent per packet, all at once. Give each this prompt:

```
<role> task: <one-line goal>

Context: <what exists, what is known, what is ruled out>
Owns (exactly these files): <paths>
Do: <concrete spec; what NOT to touch>
Accept when: <verifiable criteria>
Report back: <summary, files touched, deviations>
Do not: commit, push, or touch files outside the Owns list
```

Respect the harness's parallel limit. Batch if you have more packets than slots.

### 5. Supervise
If your harness can watch or steer running subagents, use it to correct mid-run. Otherwise wait for the results.

When results return, per packet:
- Implement: read the diff, judge against Accept-when.
- review/test findings feed the verdict.
- **GO** = done. **REJECT** = redo with a corrected packet carrying the findings and exact scope. Never rerun the identical prompt.
- Cap 3 redo rounds per packet, then surface the failure and evidence to the user.

### 6. Report
Short: what shipped (verified by your own reads), redo rounds, open questions.

## Session memory

After the user picks roles and model once, offer "reuse these picks for the rest of this session". If accepted, skip the menus on later fan-outs unless the task shape changes.

## Anti-patterns

- Asking for a model per subagent when the user wanted one uniform pick (offer per-role only as opt-in).
- Changing your own session's model. The pick is for subagents only.
- Packets like "fix the auth bug" with no file path.
- Running dependent packets in parallel (`test` needs `implement` output).
- Fanning out a task one agent could finish in a minute.
- Accepting a child's "done" without reading the diff.
