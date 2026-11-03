# Hermes Agent

Tools: `clarify` to ask (`multi_select: true` for the role menu), `delegate_task` to spawn.

## Model picker

1. Run `hermes config get delegation.model`.
2. If set, first option: **Use configured: `<value>`**.
3. If unset, say so plainly. Options: **Match my current model**, **Paste a model id**, **Set a default** (user runs `hermes config set delegation.model <id>`).

`delegate_task` has no per-call model parameter; the pin is read from `delegation.model` at dispatch. The agent cannot edit config (guarded). Ask the user to run the command, then verify with `hermes config get delegation.model`. Do not trust "done".

If the pin will not change, do not stall: use the CLI fallback below.

**Escalation** (second REJECT): the pin is global, so a stronger model for one redo means the CLI fallback with `-m <stronger-model>`, or asking the user to re-pin `delegation.model` and verifying it before the redo.

## Dispatch

One `delegate_task` call, one `tasks` entry per packet. The packet goes in each entry's `context`.

Respect `delegation.max_concurrent_children`. Batch if you have more packets than slots.

## Read-only roles

If your `delegate_task` takes a per-task toolset list, give review and explore packets no file-writing tools. Otherwise read-only is by instruction only.

## Worktrees

Children run terminal commands in your checkout. Before dispatch, create one worktree per implement packet and put its path in the packet's Checkout line:

`git worktree add -b <branch> <path> <base>`

Re-run gates in that path. After GO, merge the branch into the base, then `git worktree remove <path>` and `git branch -d <branch>`.

If children have no terminal: no worktrees. Packets keep disjoint files, make no commits, and tests run only after the whole wave returns. This fallback is weaker: parallel packets share one checkout, so one packet's half-done edit can break another's test run.

## Supervise

While children run, use `delegate_task` `action=list` to watch and `action=steer` to correct mid-run. A redo round after REJECT is a fresh child with a corrected packet, not more steering.

## CLI subagent fallback

Use only when `delegation.model` cannot be pinned and the user needs a specific model.

1. Write each packet to a scratch `.md` file.
2. Smoke-test the model with a one-line prompt:
   `hermes chat -Q --oneshot -m <model-id> --provider <provider> -q "reply ok"`
3. Run each packet in the background with notify on exit, from its worktree:
   `hermes chat -Q --oneshot --yolo -m <model-id> --provider <provider> --max-turns N --run-budget S --query-file <packet.md>`
   (terminal `background=true`, `notify_on_complete=true`)
4. Have each packet write its report to a scratch file.
5. Re-run the packet's gates and read the actual diff before GO.

`--yolo` skips approvals: only use it with disjoint file ownership and a trusted packet.
