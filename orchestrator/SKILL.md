---
name: orchestrator
description: Use when the user says "orchestrate", "fan out", "spawn agents", "split this across subagents", or gives a task with 2+ independent workstreams (research + implement + test). Asks which roles and which model to use, dispatches parallel subagents, then verifies results before reporting.
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [delegation, subagents, orchestration, parallel]
---

# Orchestrator

You stay the orchestrator. You plan, dispatch, verify, and report. Subagents do the heavy work. Requires Hermes Agent (`delegate_task`, `clarify`).

## When NOT to use

If the task is one small edit, one lookup, or one file, do it yourself and say so. Fan-out costs tokens and adds coordination overhead. Recommend the minimal set (often 1 agent, or none) before offering the full menu.

## Hard rules

1. **Ask before spawning.** Show the role menu and model picker. Skip a question only if the user already named that choice in their request, or said "same as last time" (see Session memory).
2. **Fan out in one call.** All independent packets go in a single `delegate_task` call so they run in parallel.
3. **Disjoint files.** Each packet owns its own files. If two roles need one file, one owns it; others get a handoff note.
4. **Delegate the work, keep the glue.** Orchestrator dispatches, reads results, reports. Implementation goes to implement packets. Plan files, todo bookkeeping, and user questions are yours.
5. **Self-contained packets.** Subagents cannot see this conversation. See the packet template.
6. **Verify before GO.** Subagent reports are self-reports. Read the diff or run the check yourself before accepting.

## Workflow

### 1. Scope
Restate the task in one sentence. Sketch the decomposition. If it fits one agent, say so and offer that.

### 2. Role menu
Ask with `clarify`, `multi_select: true`. Mark the suggested roles.

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

1. Run `hermes config get delegation.model`.
2. If set, first option: **Use configured: `<value>`**.
3. If unset, say so plainly. Options: **Match my current model**, **Paste a model id**, **Set a default** (user runs `hermes config set delegation.model <id>`).
4. Optional: **Per-role models** (e.g. cheap model for explore, strong model for review). Ask once per role.

`delegate_task` has no per-call model parameter; the pin is read from `delegation.model` at dispatch. The agent cannot edit config (guarded). Ask the user to run the command, then verify with `hermes config get delegation.model`. Do not trust "done".

If the pin will not change, do not stall. See `references/cli-fallback.md` to run each packet as a CLI subagent instead.

### 4. Dispatch
One `delegate_task` call, one `tasks` entry per packet. Put this in each `context`:

```
<role> task: <one-line goal>

Context: <what exists, what is known, what is ruled out>
Owns (exactly these files): <paths>
Do: <concrete spec; what NOT to touch>
Accept when: <verifiable criteria>
Report back: <summary, files touched, deviations>
Do not: commit, push, or touch files outside the Owns list
```

Respect the user's parallel-child limit (`delegation.max_concurrent_children`). Batch if you have more packets than slots.

### 5. Supervise
While children run, use `delegate_task` `action=list` to watch and `action=steer` to correct mid-run.

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
- Packets like "fix the auth bug" with no file path.
- Running dependent packets in parallel (`test` needs `implement` output).
- Fanning out a task one agent could finish in a minute.
- Accepting a child's "done" without reading the diff.
