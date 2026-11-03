# Claude Code and Rayu

Tools: `AskUserQuestion` to ask, `Agent` to spawn (named `Task` in older versions). Rayu uses the same tools.

## Asking

`AskUserQuestion` takes 2–4 options per question and always adds "Other" for free text. Use `multiSelect: true` for the role menu. Put the suggested roles first and mark them "(Recommended)".

## Model picker

The spawn tool has a `model` parameter, set per call.

- **Use default**: omit `model`. The subagent uses its agent definition's model, or the session's.
- **Match my current model**: pass the session's model explicitly.
- **Paste a model id**: pass it unchanged in `model`. Claude Code accepts the aliases `opus`, `sonnet`, `haiku`; other values depend on the user's provider setup.
- **Per-role**: a different `model` per role; every packet of that role carries it.
- **Escalation** (second REJECT): pass the stronger model in `model` on the fresh redo.

If the user has agent definitions (`~/.claude/agents/*.md`, or `.claude/agents/` in the project) whose models fit the roles, offer them by name as `subagent_type`.

## Read-only roles

`general-purpose` has every tool; only an agent definition's frontmatter limits them. Suggest the user define read-only agents once, e.g. `~/.claude/agents/reviewer.md`:

```markdown
---
name: reviewer
description: Read-only reviewer. Gets the spec and the diff; reports findings, never edits.
disallowedTools: Write, Edit, NotebookEdit
---
```

Use it as `subagent_type` for review packets, and a similar `explorer` for explore. Bash can still write files, so the packet's "no edits" still applies. Agent definitions load when a session starts. Without them, review and explore are read-only by instruction only.

## Dispatch

One `Agent` call per packet, and every packet of the wave in one assistant message, so they run in parallel: three implement units are three `Agent` calls. Per call:

- `subagent_type`: `general-purpose` for implement and test. A read-only agent (above) for review, else `general-purpose`. `Explore` or a read-only agent for explore.
- `description`: 3–5 words.
- `prompt`: the packet.
- `model`: per the picker, or omitted.
- `isolation: "worktree"` on every implement packet.

## Worktrees

`isolation: "worktree"` gives the subagent its own checkout on its own branch. It branches from the session's current HEAD, so name the base branch in the packet's Base line. The result names the worktree path and branch: re-run gates there. After GO, merge the branch into the base, then `git worktree remove <path>` and `git branch -d <branch>`. A worktree left unchanged is cleaned up automatically.

If your version has no `isolation` parameter (check Rayu), create worktrees yourself with `git worktree add -b <branch> <path> <base>` and put the path in the packet's Checkout line.

## Supervise

Results come back when each subagent finishes. To correct one, spawn a corrected packet in a fresh subagent. Continue a finished agent (`SendMessage`) only for a quick follow-up while its context is small, never as a redo round.
