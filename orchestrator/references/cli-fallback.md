# CLI subagent fallback

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
