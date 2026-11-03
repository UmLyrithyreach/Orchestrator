# Antigravity CLI (agy)

Tools: `ask_question` to ask, `invoke_subagent` to spawn, `define_subagent` to create a reusable subagent type. If your tool list names them differently, use the tool that does that job.

## Asking

Use `ask_question` for the role menu and the model picker, allowing several answers for roles. If it is not available, ask in plain text.

## Model picker

`invoke_subagent` has no model parameter. Subagents run on the session's model or on Antigravity's own subagent tier, depending on the user's settings.

- **Use default / Match my current model**: dispatch as is.
- **Paste a model id**: say plainly that a subagent's model can't be set per call. The user can start the session on that model (`agy --model <id>`; `agy models` lists ids), which subagents then inherit. Never claim a model you did not set.
- **Escalation** (second REJECT): no stronger model per call either. Say so, and ask once: restart the session on a stronger model, or stop and show the evidence.

## Read-only roles

`define_subagent` once per read-only role (reviewer, explorer) with no file-editing tools, then invoke it by name for review and explore packets. Shell access can still write files, so the packet's "no edits" still applies.

## Dispatch

One `invoke_subagent` call with every packet in its `subagents` list, so they run in parallel. Each entry gets a short name and the packet as its prompt.

## Worktrees

Subagents share the session's checkout. If they have a shell, create one worktree per implement packet before dispatch and put its path in the packet's Checkout line:

`git worktree add -b <branch> <path> <base>`

Re-run gates in that path. After GO, merge the branch into the base, then `git worktree remove <path>` and `git branch -d <branch>`.

If they have no shell: no worktrees. Packets keep disjoint files, make no commits, and tests run only after the whole wave returns. This fallback is weaker: parallel packets share one checkout, so one packet's half-done edit can break another's test run.

## Supervise

The user can watch subagents in the `/agents` panel. Subagent tool calls may wait for the user's approval there; if a result is slow, tell the user to check `/agents` for a pending approval. To correct a finished subagent, send it a message by its conversation id only for a quick follow-up; a redo round is always a fresh subagent with a corrected packet.
