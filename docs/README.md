# Documentation map

Use this page after `README.md`, `AGENTS.md`, `STATE.md` and `TASKS.md`.

## Current subsystem authority

- `architecture/` — ownership, storage, lifecycle, ABI and reverse-analysis boundaries.
- `hardware/` — board, boot and platform facts.
- `sensor/` — current GC1054 and dual-sensor/lens-switch contracts.
- `isp/` — current AE/AWB/color/LTM/WDR/NR3D findings and boundaries.
- `media/` — VI/VENC/media/RPC/timing/JPEG/OSD findings.
- `audio/` — current audio ABI and stock/vendor observations.
- `ptz/` — accepted `/dev/fh_pwm` backend and remaining integration work.
- `streamers/` — Majestic and Divinus integration state. Camera-level facts remain outside streamer-specific files.
- `process/` — current acceptance, build/flash, operator-safety and related-repository integration rules.

## OpenIPC integration rules

Before modifying or preparing contributions for related OpenIPC repositories, read:

- `process/openipc-upstream-rules.md` — cached current OpenIPC ownership/contribution rules plus mandatory live source links and refresh procedure;
- `process/upstream-integration.md` — this project's current FH8626 repository refs, preservation checkpoints, ownership routing and work order.

`openipc-upstream-rules.md` is intentionally not a substitute for upstream documentation. Agents must open the live links recorded there before implementation/contribution work and update the local summary if OpenIPC rules have changed.

## Reverse analysis

Active reverse work does not live in a Git directory. The canonical working reverse environment is Ghidra through Koba MCP Bridge / Ghidra MCP.

Start with `architecture/reverse-analysis.md` for the authority boundary and promotion rules.

Do not recreate historical local Ghidra project paths, Java helper sets, generated export trees or a `reverse/` scratch hierarchy from old documentation or Git history.

## Other top-level areas

- `source/` — retained target-specific source, contracts and tests.
- `evidence/` — Git-side manifest for heavy/unique evidence retained on Google Drive.
- `state/` — compact machine-readable facts, hypotheses and decisions.
- `history/` — historical chronology and provenance. It is not current authority.

## Current-vs-history rule

For present project state, prefer `STATE.md`, `TASKS.md` and the current `docs/` tree. Historical files may explain how a conclusion was reached, but labels such as `current`, `latest` or `canonical` inside historical material are relative to their original date and do not override current documentation.
