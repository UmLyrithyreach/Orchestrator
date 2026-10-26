# Antigravity CLI (agy)

Tools: `ask_question` to ask, `invoke_subagent` to spawn, `define_subagent` to create a reusable subagent type. If your tool list names them differently, use the tool that does that job.

## Asking

Use `ask_question` for the role menu and the model picker, allowing several answers for roles. If it is not available, ask in plain text.

## Model picker

`invoke_subagent` has no model parameter. Subagents run on the session's model or on Antigravity's own subagent tier, depending on the user's settings.

- **Use default / Match my current model**: dispatch as is.
- **Paste a model id**: say plainly that a subagent's model can't be set per call. The user can start the session on that model (`agy --model <id>`; `agy models` lists ids), which subagents then inherit. Never claim a model you did not set.

## Dispatch

One `invoke_subagent` call with every packet in its `subagents` list, so they run in parallel. Each entry gets a short name and the packet as its prompt.

For a role you will reuse (e.g. a read-only reviewer), `define_subagent` once with only the tools it needs, then invoke it by name.

## Supervise

The user can watch subagents in the `/agents` panel. Subagent tool calls may wait for the user's approval there; if a result is slow, tell the user to check `/agents` for a pending approval. To correct a finished subagent, send it a message by its conversation id only for a quick follow-up; otherwise spawn a corrected packet.
