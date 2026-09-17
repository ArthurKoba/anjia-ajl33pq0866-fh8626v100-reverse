# Structured current state

This directory is a compact machine-readable snapshot of the current project boundary:

- `facts.jsonl` — current supported facts;
- `hypotheses.jsonl` — open/deferred claims with their next validation step;
- `decisions.jsonl` — current architecture/process decisions.

For human-readable authority use `STATE.md`, `TASKS.md` and the current subsystem documents under `docs/`.

When a fact or decision materially changes, update the relevant human-readable authority and then update this snapshot. Historical chronology belongs under `history/`.
