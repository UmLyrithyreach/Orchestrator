---
name: orchestrator
description: Use when the user says "orchestrate", "fan out", "spawn agents", "split this across subagents", or gives a task with 2+ independent workstreams (research + implement + test). Asks which roles and which model to use, dispatches parallel subagents, then verifies each result and approves it or sends it back for a redo. The main model manages and never does the implementation itself.
version: 1.3.0
license: MIT
compatibility: Hermes Agent, Claude Code, OpenCode, Rayu, Antigravity CLI
metadata:
  hermes:
    tags: [delegation, subagents, orchestration, parallel]
---

# Orchestrator

You stay the orchestrator: the manager, not a worker. You plan, dispatch, verify, and report. Subagents do the heavy work in their own contexts, so yours stays small.

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
2. **Fan out within each role, not just across roles.** A role is a kind of work; a packet is one unit of it. A role whose work splits into N independent units gets N packets, each its own subagent on that role's model and subagent type. Picking implement + test never means "one implementer and one tester" unless each really is a single unit.
3. **Fan out in one step.** All independent packets of a wave go out together so they run in parallel: one `delegate_task` or `invoke_subagent` call, or several spawn calls in one message.
4. **Disjoint files.** Each packet owns its own files. If two roles need one file, one owns it; others get a handoff note.
5. **Delegate the work, keep the glue.** Orchestrator dispatches, reads results, reports. While orchestrating, never edit project files or write code yourself: implementation goes to implement packets. If a result is wrong, REJECT it and send a corrected packet; do not patch it yourself. Plan files, todo bookkeeping, and user questions are yours.
6. **Self-contained packets.** Subagents cannot see this conversation. See the packet template.
7. **Verify before GO.** Subagent reports are self-reports. Read the diff (not whole files) and the test packet's evidence; run a check yourself only when no test packet covers it. Verification is the only heavy reading you do, so keep it to what the Accept-when criteria need.

## Workflow

### 1. Scope
Restate the task in one sentence. Sketch the decomposition. If it fits one agent, say so and offer that.

Once roles are picked, split each role into its units and show the plan as roles × packets before dispatch, e.g. `implement ×3 (limiter middleware, config, route wiring) · test ×2 (unit, integration)`. Units must be independent and own disjoint files; work that can't be split stays one packet.

### 2. Role menu
**Ask**, multi-select. Mark the suggested roles.

| Role | Does |
|---|---|
| implement | Writes the changes. One packet per independent unit (module, file group, endpoint). |
| review | Critiques plan or output. Findings only, no edits. One packet per GO'd implement unit or review lens. |
| test | Writes or runs tests. Reports pass/fail with evidence. One packet per unit or suite under test. |
| explore | Recon first, so implement packets get exact paths and line numbers. One packet per unknown area. |

The user can type a custom role via "Other" (e.g. docs, security-audit, migrate). Treat it as another role: split it into units, one packet each, with its own Owns/Accept criteria.

Sequencing runs in waves: `explore` packets, then all `implement` packets, then `review` and `test` packets. Each wave fans out in full. A wave-3 packet starts as soon as the implement unit it covers is GO; it doesn't wait for sibling units.

### 3. Model picker
Never assume a model. Every user has different providers.

Each harness sets the subagent model differently; your reference says how. Offer, in this order:

1. **Use configured / default**: the subagent model already set up (name it if you can read it).
2. **Match my current model**: the session's model, passed explicitly.
3. **Paste a model id**: the user types the exact id; pass it unchanged. If it looks like an alias or partial name, resolve it and confirm once.
4. Optional: **Per-role models** (e.g. cheap model for explore, strong model for review). Ask once per role. Every packet of a role uses that role's pick.

If the harness cannot apply the pick, say so plainly and offer what it can do. Never claim a model you did not set.

### 4. Dispatch
**Spawn** one subagent per packet, every packet of the wave at once. Three implement units means three implement subagents in the same dispatch, not one implementer handed all three. Give each this prompt:

```
<role> task: <one-line goal>

Context: <what exists, what is known, what is ruled out>
Owns (exactly these files): <paths>
Do: <concrete spec; what NOT to touch>
Accept when: <verifiable criteria>
Report back: <summary, files touched, deviations>
Do not: commit, push, or touch files outside the Owns list
```

Respect the harness's parallel limit. Batch if you have more packets than slots, and say the batching.

### 5. Supervise
If your harness can watch or steer running subagents, use it to correct mid-run. Otherwise wait for the results.

You watch over every packet of the wave, not the role as a whole. When results return, per packet:
- Implement: read the diff, judge against Accept-when.
- review/test findings feed the verdict.
- **GO** = done. **REJECT** = redo with a corrected packet carrying the findings and exact scope. Never rerun the identical prompt.
- A REJECT redoes only that packet; its GO'd siblings stand and their next-wave packets proceed.
- Cap 3 redo rounds per packet, then surface the failure and evidence to the user.

### 6. Report
Short: per role, how many packets ran and their verdicts; what shipped (verified by your own reads), redo rounds, open questions.

## Session memory

After the user picks roles and model once, offer "reuse these picks for the rest of this session". If accepted, skip the menus on later fan-outs unless the task shape changes.

## Anti-patterns

- Asking for a model per subagent when the user wanted one uniform pick (offer per-role only as opt-in).
- Changing your own session's model. The pick is for subagents only.
- One packet per role when the role's work splits into independent units: one implementer doing three modules in series.
- Packets like "fix the auth bug" with no file path.
- Running dependent packets in parallel (`test` needs `implement` output).
- Fanning out a task one agent could finish in a minute.
- Accepting a child's "done" without reading the diff.
