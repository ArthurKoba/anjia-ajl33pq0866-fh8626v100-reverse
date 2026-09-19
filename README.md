# ANJIA AJL33PQ0866 / FH8626V100

Camera-level engineering repository for the ANJIA AJL33PQ0866 based on Fullhan FH8626V100.

This repository owns durable camera contracts, current engineering state, camera-specific components, integration knowledge and the external-evidence index.

## Hardware

- SoC: FH8626V100
- Camera: ANJIA AJL33PQ0866
- Sensors: dual GC1054 MIPI
- Exercised native mode: 1280x720 @ 25 fps
- Product default lens: WIDE
- PTZ backend: `/dev/fh_pwm`

## Current direction

Majestic is the preferred product path. Divinus remains an open reference and diagnostic implementation.

Camera-level facts stay independent of either streamer. Sensor sequencing, ISP behavior, media ownership, audio, PTZ and boot contracts belong here; implementation changes belong in the repository that owns the relevant OpenIPC component.

Reverse engineering is performed through the canonical Ghidra project exposed by Ghidra MCP through Koba MCP Bridge. Durable conclusions are promoted into the appropriate Git documentation or reusable source contract.

## Authority model

- **GitHub** — current source, documentation, contracts, state and evidence manifests.
- **Google Drive evidence store** — firmware, dumps, retained vendor binaries, captures and other heavy or unique primary evidence, addressed from Git by SHA-256.
- **Ghidra MCP** — mutable reverse-analysis workspace and working reverse index.

Generated reverse output is not an authority by itself; primary bytes remain in the evidence store and durable technical conclusions remain in Git.

## Current engineering state

- Dual GC1054 operation and the board-level cold-boot/lens-switch requirements are documented.
- `/dev/fh_pwm` PTZ control is hardware-proven on the exercised board.
- Curated camera-specific components and contracts are retained under `source/fh8626v100/components/` for later integration into their owning repositories.
- Majestic can start with the retained proprietary Fullhan stack; reproducible VI -> VENC -> sustained RTSP acceptance remains the main product-validation boundary.
- Divinus native FH8626 work remains useful as a reference path but is not the preferred product baseline.

See `STATE.md` for the exact current boundary and `TASKS.md` for actionable work.

## Repository layout

- `docs/` — current subsystem and architecture documentation.
- `source/` — reusable camera-specific components, contracts and tests.
- `evidence/` — manifest for external primary evidence.
- `state/` — compact machine-readable facts, hypotheses and durable decisions.
- `history/` — engineering chronology, evidence provenance and durable lessons.

## Related repositories

Implementation work is distributed across:

- `ArthurKoba/openipc-builder`
- `ArthurKoba/openipc-divinus`
- `ArthurKoba/openipc-firmware`
- `ArthurKoba/openipc-linux`
- `ArthurKoba/u-boot-fullhan`

Exact engineering refs are recorded in `docs/process/upstream-integration.md`.

## Start here

1. `AGENTS.md` — mandatory repository map/router.
2. `STATE.md` — current camera state.
3. `TASKS.md` — actionable work.
4. `ROADMAP.md` — staged project direction.
5. `docs/README.md` — subsystem documentation map.
6. `docs/architecture/reverse-analysis.md` — Ghidra MCP reverse boundary.
7. `docs/process/openipc-upstream-rules.md` — cached OpenIPC ownership/contribution rules, mandatory live-source links and refresh procedure before related-repository work.
8. `evidence/README.md` and `evidence/MANIFEST.tsv` — external evidence rules and index.
9. `docs/process/agent-operation.md` — project-specific Git/tool/build/coordination policy.

Agents must not create pull requests. See `AGENTS.md`.
