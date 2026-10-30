---
name: orchestrator
description: Use when the user says "orchestrate", "fan out", "spawn agents", "split this across subagents", or gives a task with 2+ independent workstreams (research + implement + test). Asks which roles and which model to use, dispatches parallel subagents, then re-runs each result's gates and approves it or sends it back for a redo. The main model manages and never does the implementation itself.
version: 1.4.0
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
4. **Disjoint files, separate checkouts.** Each packet owns its own files. If two roles need one file, one owns it; others get a handoff note. Each implement packet works in its own git worktree on its own branch and commits there; your reference says how, and what to do when subagents have no shell. Keep files disjoint even with worktrees: it prevents merge conflicts.
5. **Delegate the work, keep the glue.** Orchestrator dispatches, reads results, re-runs gates, merges, reports. While orchestrating, never edit project files or write code yourself: implementation goes to implement packets. If a result is wrong, REJECT it and send a corrected packet. One exception: if a REJECT comes down to a fix of a few lines, and the corrected packet would take more text than the fix, make it yourself, re-run that packet's gates, and note it in the report. Everything bigger goes back out as a packet. Plan files, todo bookkeeping, and user questions are yours.
6. **Self-contained packets.** Subagents cannot see this conversation. See the packet template.
7. **Verify before GO.** Every subagent report is a self-report, a test packet's included. Before GO, re-run every runnable gate yourself, in the packet's checkout: it takes seconds and replaces trusting the report. Read the diff (not whole files) for what the gates can't see. MANUAL gates go to a review packet. Verification is the only heavy reading you do, so keep it to what the gates need.

## Workflow

### 1. Scope
Restate the task in one sentence. Sketch the decomposition. If it fits one agent, say so and offer that.

Once roles are picked, split each role into its units and show the plan as roles × packets before dispatch, e.g. `implement ×3 (limiter middleware, config, route wiring) · test ×2 (unit, full suite)`. Units must be independent and own disjoint files; work that can't be split stays one packet.

### 2. Role menu
**Ask**, multi-select. Mark the suggested roles.

| Role | Does |
|---|---|
| implement | Writes the changes on its own branch. Runs only the narrow tests that cover its change. One packet per independent unit (module, file group, endpoint). |
| review | Critiques output against the spec. Gets only the original requirement and the diff (or branch/commit range): never the plan, the implementer's report or its reasoning. Findings only, no edits. Decides MANUAL gates. One packet per implement unit or review lens. |
| test | Writes tests, one packet per unit under test. The full suite runs once, after the wave's implement branches are merged, in one test packet on the cheapest model. Reports pass/fail with evidence. |
| explore | Recon first, so implement packets get exact paths and line numbers. One packet per unknown area. |

The user can type a custom role via "Other" (e.g. docs, security-audit, migrate). Treat it as another role: split it into units, one packet each, with its own Owns list and gates.

review and explore are read-only. Where the harness can enforce that with per-agent tool limits, use it; your reference says how. Elsewhere read-only is by instruction only.

Sequencing runs in waves: `explore` packets, then all `implement` packets, then `review` and `test` packets. Each wave fans out in full. A review packet starts as soon as the implement unit it covers passes its gates; it doesn't wait for sibling units. The full-suite test packet waits until every implement branch of the wave is merged.

### 3. Model picker
Never assume a model. Every user has different providers.

Each harness sets the subagent model differently; your reference says how. Offer, in this order:

1. **Use configured / default**: the subagent model already set up (name it if you can read it).
2. **Match my current model**: the session's model, passed explicitly.
3. **Paste a model id**: the user types the exact id; pass it unchanged. If it looks like an alias or partial name, resolve it and confirm once.
4. Optional: **Per-role models** (e.g. cheap model for explore and test, strong model for review). Ask once per role. Every packet of a role uses that role's pick.

If the harness cannot apply the pick, say so plainly and offer what it can do. Never claim a model you did not set.

### 4. Dispatch
**Spawn** one subagent per packet, every packet of the wave at once. Three implement units means three implement subagents in the same dispatch, not one implementer handed all three. Give each this prompt:

```
<role> task: <one-line goal>

Context: <what exists, what is known, what is ruled out>
Base: <branch to start from>  Branch: <your branch>  Checkout: <worktree path, if given>
Owns (exactly these files): <paths>
Do: <concrete spec; what NOT to touch>
Gates (one per required outcome):
  G1 <outcome> — CHECK: <command> — EXPECT: <text the output must contain>
  G2 <outcome> — MANUAL
Report back: each gate's result in one line, files touched, deviations
Do not: touch files outside the Owns list. Commit only on your own branch; never push or merge.
```

- **One gate per required outcome.** A green suite doesn't prove every outcome exists. An outcome no command can decide is `MANUAL` and goes to a review packet.
- **Negative gate for fixes.** When a packet fixes a bug and a test is said to cover the fix, add a gate that sets the fix aside, runs that test, expects it to FAIL, then restores the fix:
  `git diff <base> -- <fix paths, not test paths> > <scratch>/fix.patch && git apply -R <scratch>/fix.patch` → run the test (expect failure) → `git apply <scratch>/fix.patch`.
  `git add -N` new files first. Use the patch file, never `git stash`: the stash is shared by every worktree. A test that passes either way proves nothing; a run where zero tests executed is not a pass.
- **Review packets** carry the original requirement and the diff or branch/commit range, nothing else.
- **Implement packets** run only the narrow tests covering their change. The full suite is the test packet's job, after merge.

Respect the harness's parallel limit. Batch if you have more packets than slots, and say the batching.

### 5. Supervise
If your harness can watch or steer running subagents, use it to correct mid-run. Otherwise wait for the results.

You watch over every packet of the wave, not the role as a whole. When results return, per packet:
- Re-run its runnable gates yourself; read the diff for what they can't see.
- MANUAL gates and review/test findings feed the verdict.
- **GO** = done. Merge its branch into the base, then remove its worktree and branch.
- A REJECT redoes only that packet; its GO'd siblings stand and their next-wave packets proceed.
- A red full suite goes to a fresh implement packet carrying the failures.

On REJECT, escalate the model instead of repeating:
1. **First REJECT**: a corrected packet, carrying the findings and exact scope, to a fresh subagent on the same model.
2. **Second REJECT**: redo the packet in a fresh subagent on a stronger model: the review role's pick if it is stronger, or ask once.
3. **Still failing**: stop. Show the user the packet, the gate results, and the evidence.

Never rerun the identical prompt. Never keep coaching one subagent across rounds: a resumed agent re-sends its whole history every turn.

### 6. Report
Short: per role, how many packets ran and their verdicts; what shipped (verified by your own gate re-runs), REJECTs and escalations, small fixes you made yourself, open questions.

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
- Accepting a test packet's "all green" without re-running the gates yourself.
- Handing the reviewer the implementer's explanation: the plan, its report or its reasoning.
- A fix gate that can't fail: the test passes with the fix set aside, or zero tests ran.
- Setting a fix aside with `git stash` instead of a patch file.
- Parallel implementers sharing one checkout when worktrees are available.
- Every implement packet running the full suite.
- Coaching one subagent across rounds, or a third redo on the same model, instead of escalating.
