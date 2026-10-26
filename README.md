# orchestrator

A [Hermes Agent](https://hermes-agent.nousresearch.com) skill that fans a task out to parallel subagents, and lets **you** choose the roles and the models.

Most delegation skills hardcode a subagent model. Yours won't match mine. This one asks.

## What it does

1. Scopes the task. Tells you if one agent is enough.
2. Asks which roles to run: implement, review, test, explore, or your own.
3. Reads your `delegation.model`, then asks which model the subagents should use (configured, current, pasted id, or per-role).
4. Dispatches all packets in one parallel `delegate_task` call.
5. Checks every result against its acceptance criteria. Rejects and redoes up to 3 times. Reports.

## Install

```bash
git clone https://github.com/UmLyrithyreach/Orchestrator.git
cp -r Orchestrator/orchestrator ~/.hermes/skills/
```

Start a new Hermes session and say: `orchestrate: add dark mode to the settings page`.

## Example

```
> orchestrate: add rate limiting to the API and cover it with tests

Scope: 2 independent parts (code, tests). Suggest: explore, implement, test.
? Roles  [x] explore  [x] implement  [ ] review  [x] test
? Model  > Use configured: my-fast-model
Dispatching 1 explore packet... done. Dispatching implement + test...
implement: GO (diff read). test: 14/14 pass. Shipped.
```

## Requirements

- Hermes Agent with `delegate_task` and `clarify` tools.
- Optional: `hermes config set delegation.model <id>` to pin the subagent model.

## Files

- `orchestrator/SKILL.md` the skill
- `orchestrator/references/cli-fallback.md` run packets as CLI subagents when the model pin is blocked

## License

MIT
