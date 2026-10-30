# OpenCode

Tools: `question` to ask, `task` to spawn.

## Asking

`question` takes a list of questions; set `multiple: true` for the role menu. It is on in the TUI, desktop and app clients. Elsewhere (e.g. `opencode run`) it may be missing unless `OPENCODE_ENABLE_QUESTION_TOOL=1`; then ask in plain text.

## Model picker

`task` has no model parameter. The subagent's model comes from its agent definition (`subagent_type`). With no `model` set there, it uses the session's model.

1. Check which subagents exist and their models: the `task` tool lists them, and the user can run `opencode debug agent <name>`.
2. Offer: **Use default** (`general`), **Match my current model** (also `general`, when it has no model of its own), or **an existing subagent** whose model fits.
3. For a pasted model id with no matching agent, ask the user to add one, then restart OpenCode. For example `~/.config/opencode/agent/fast.md`:

   ```markdown
   ---
   description: Fast general subagent
   mode: subagent
   model: <provider>/<model-id>
   ---
   ```

   Verify with `opencode debug agent fast` before dispatch. Do not trust "done".

**Escalation** (second REJECT): redo on an existing subagent with a stronger model. If none exists, ask once for one (as above), or stop and show the evidence.

## Read-only roles

Agent definitions can limit tools. Suggest a read-only reviewer, e.g. `~/.config/opencode/agent/reviewer.md`:

```markdown
---
description: Read-only reviewer. Gets the spec and the diff; reports findings, never edits.
mode: subagent
permission:
  edit: deny
---
```

Use it as `subagent_type` for review packets, and `explore` (or a similar read-only agent) for explore. Bash can still write files, so the packet's "no edits" still applies.

## Dispatch

All `task` calls in one message, so they run in parallel. Per call:

- `subagent_type`: per the picker (`general`, `explore`, `reviewer`, or a user agent).
- `description`: 3–5 words.
- `prompt`: the packet.

## Worktrees

Subagents share the session's checkout. Before dispatch, create one worktree per implement packet and put its path in the packet's Checkout line:

`git worktree add -b <branch> <path> <base>`

Re-run gates in that path. After GO, merge the branch into the base, then `git worktree remove <path>` and `git branch -d <branch>`.

If the subagent has no shell: no worktrees. Packets keep disjoint files, make no commits, and tests run only after the whole wave returns. This fallback is weaker: parallel packets share one checkout, so one packet's half-done edit can break another's test run.

## Supervise

Results come back when each subagent finishes. To correct one, spawn a corrected packet. Pass `task_id` to continue a finished subagent only for a quick follow-up, never as a redo round.
