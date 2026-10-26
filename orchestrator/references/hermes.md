# Hermes Agent

Tools: `clarify` to ask (`multi_select: true` for the role menu), `delegate_task` to spawn.

## Model picker

1. Run `hermes config get delegation.model`.
2. If set, first option: **Use configured: `<value>`**.
3. If unset, say so plainly. Options: **Match my current model**, **Paste a model id**, **Set a default** (user runs `hermes config set delegation.model <id>`).

`delegate_task` has no per-call model parameter; the pin is read from `delegation.model` at dispatch. The agent cannot edit config (guarded). Ask the user to run the command, then verify with `hermes config get delegation.model`. Do not trust "done".

If the pin will not change, do not stall: use the CLI fallback below.

## Dispatch

One `delegate_task` call, one `tasks` entry per packet. The packet goes in each entry's `context`.

Respect `delegation.max_concurrent_children`. Batch if you have more packets than slots.

## Supervise

While children run, use `delegate_task` `action=list` to watch and `action=steer` to correct mid-run.

## CLI subagent fallback

Use only when `delegation.model` cannot be pinned and the user needs a specific model.

1. Write each packet to a scratch `.md` file.
2. Smoke-test the model with a one-line prompt:
   `hermes chat -Q --oneshot -m <model-id> --provider <provider> -q "reply ok"`
3. Run each packet in the background with notify on exit:
   `hermes chat -Q --oneshot --yolo -m <model-id> --provider <provider> --max-turns N --run-budget S --query-file <packet.md>`
   (terminal `background=true`, `notify_on_complete=true`)
4. Have each packet write its report to a scratch file.
5. Review that file and the actual diff before GO.

`--yolo` skips approvals: only use it with disjoint file ownership and a trusted packet.
