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

## Dispatch

All `task` calls in one message, so they run in parallel. Per call:

- `subagent_type`: per the picker (`general`, `explore`, or a user agent).
- `description`: 3–5 words.
- `prompt`: the packet.

## Supervise

Results come back when each subagent finishes. To correct one, spawn a corrected packet. Pass `task_id` to continue a finished subagent only for a quick follow-up.
