# Agent rules

These rules apply to every automated agent or assistant working in this repository.

## Git workflow

- Agents MUST NOT create pull requests. Pull requests are created manually by the repository owner only.
- Use a dedicated working branch for non-trivial source, documentation, reverse or cleanup changes.
- Do not make normal project changes directly on `main`.
- After verification, merge the working branch directly by normal fast-forward/merge semantics.
- Never force-update a shared branch unless the repository owner explicitly requests it.
- Upstream contribution series must be curated deliberately from a verified upstream base with coherent commits.

## Project authority

This camera project has three storage roles:

- GitHub is the authority for current camera-level source, documentation, contracts, state and evidence manifests.
- Google Drive stores heavy or unique primary evidence such as firmware, flash/rootfs/kernel/RAM dumps, retained vendor binaries and raw captures.
- Ghidra MCP, exposed through Koba MCP Bridge, is the canonical mutable reverse-analysis workspace.

## Reverse engineering

All active reverse work for this camera is performed through the canonical Ghidra MCP project.

Use Ghidra MCP to inspect functions, types, xrefs, strings, data, instructions and decompiler output. Promote only durable camera knowledge into Git: subsystem findings, useful addresses, ABI/layout conclusions, source-level reference implementations and concise reproducible contracts.

Do not commit Ghidra databases, bulk decompiler/disassembly exports or other generated reverse substrates. The Ghidra project is working analysis state; Git is the durable engineering record.

Cross-Fullhan material may be used as a semantic reference, but it never establishes an FH8626 ABI, register map, layout or runtime contract without FH8626 target evidence.

Broad reverse is not a standing task. Re-enter reverse only for a concrete implementation, validation or contradictory-evidence question.

## Evidence

Keep heavy primary evidence out of Git unless there is a specific documented reason otherwise. Every externally retained object used by current work should have:

- SHA-256;
- role/provenance;
- durable external locator;
- enough notes to distinguish primary bytes from generated derivatives.

Record these in `evidence/MANIFEST.tsv`.

## Evidence classes

Use explicit evidence boundaries when they matter:

- `HARDWARE_PASS`
- `STOCK_RUNTIME`
- `REVERSE_CONFIRMED`
- `SOURCE_CONFIRMED`
- `OBSERVATION`
- `HYPOTHESIS`
- `SUPERSEDED`

Do not silently promote static/source/reverse coverage into hardware acceptance.

## Documentation

- Keep one current authority for each fact or contract.
- Current subsystem facts belong under `docs/`.
- Historical chronology, provenance and durable lessons belong under `history/`.
- Camera-level sensor, ISP, media, audio, PTZ and boot knowledge must remain independent of a particular streamer.
- Keep current documentation focused on present architecture and actionable engineering boundaries.

## Source

`source/` contains target-specific camera engineering material that is useful to preserve or validate. It is not automatically upstream-ready code.

Do not commit build outputs, caches or generated analysis as source. Keep source-level tests and contracts when they materially describe or validate retained implementation behavior.

## Related repositories

Changes to related repositories such as `openipc-divinus`, `openipc-builder`, `openipc-firmware`, `openipc-linux` and `u-boot-fullhan` must also use working branches. The no-agent-PR rule applies there as well.

Do not duplicate camera-level knowledge into those repositories. Record new camera contracts here, then implement or reference them in the repository that owns the component.
