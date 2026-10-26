# Claude Code and Rayu

Tools: `AskUserQuestion` to ask, `Agent` to spawn (named `Task` in older versions). Rayu uses the same tools.

## Asking

`AskUserQuestion` takes 2–4 options per question and always adds "Other" for free text. Use `multiSelect: true` for the role menu. Put the suggested roles first and mark them "(Recommended)".

## Model picker

The spawn tool has a `model` parameter, set per call.

- **Use default**: omit `model`. The subagent uses its agent definition's model, or the session's.
- **Match my current model**: pass the session's model explicitly.
- **Paste a model id**: pass it unchanged in `model`. Claude Code accepts the aliases `opus`, `sonnet`, `haiku`; other values depend on the user's provider setup.
- **Per-role**: a different `model` on each call.

If the user has agent definitions (`~/.claude/agents/*.md`, or `.claude/agents/` in the project) whose models fit the roles, offer them by name as `subagent_type`.

## Dispatch

All spawn calls in one assistant message, so they run in parallel. Per call:

- `subagent_type`: `general-purpose` for implement, review and test. `Explore` for explore when it is read-only recon.
- `description`: 3–5 words.
- `prompt`: the packet.
- `model`: per the picker, or omitted.

When parallel makers could touch the same checkout, pass `isolation: "worktree"` if the harness offers it.

## Supervise

Results come back when each subagent finishes. To correct one, spawn a corrected packet; continue a finished agent (`SendMessage`) only for a quick follow-up while its context is small.
