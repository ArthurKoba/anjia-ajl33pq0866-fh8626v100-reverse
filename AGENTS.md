# Agent rules

These rules apply to every automated agent or assistant working in this repository.

## Git workflow

- Agents MUST NOT create pull requests. Pull requests are created manually by the repository owner only.
- Keep the branch layout easy to understand. A repository should normally have a clear production/integration branch and a clear current development branch.
- Topic branches are fine for substantial or risky work when isolation is useful. Do not create a separate branch for every small edit, audit note or micro-fix.
- Once a topic branch has been checked and its work belongs in the current development line, merge/fast-forward it into that development branch and remove the now-redundant topic branch.
- Do not leave completed work scattered across several live branches. At any point, another agent should be able to identify the current production state and the current development state without reconstructing branch history.
- Production/integration branches receive state only after the applicable verification gate. Unverified work belongs in the current development line or, when genuinely useful, a short-lived topic branch.
- Historical checkpoints and superseded experiments should normally be retained as annotated tags, immutable commit SHAs or evidence rather than long-lived active branches.
- After verification, promote the development state by normal fast-forward/merge semantics where the repository model permits it.
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

A SHA that appears only in `history/` and has no current manifest row is provenance-only. Do not treat it as a current retrievable dependency until the object is re-located/retained and indexed.

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
- Cross-repository coordination must be synchronized in the same work session as material implementation changes. If a related repository changes an active SHA, branch topology, ownership boundary, named runtime target, build composition model or hardware-validation gate, update `STATE.md`, `TASKS.md` and the applicable `docs/process/` / subsystem document here before considering the handoff complete. Do not leave later agents to discover a newer component state than the coordination authority describes.
- Current subsystem facts belong under `docs/`.
- Historical chronology, provenance and durable lessons belong under `history/`.
- Camera-level sensor, ISP, media, audio, PTZ and boot knowledge must remain independent of a particular streamer.
- Keep current documentation focused on present architecture and actionable engineering boundaries.

## Source

`source/fh8626v100/components/` contains curated, reusable camera-specific components, contracts and source-level tests. It is not a monolithic firmware/application tree and is not automatically upstream-ready code.

Do not retain incomplete owner snapshots, generated reverse reconstructions, probes, build outputs, caches or generated analysis as current source. Keep a component only when it is independently useful for reuse, integration or direct source-level validation.

Run the retained host/source checks with:

`make -C source/fh8626v100/components check`

Host/source checks never imply target hardware acceptance.

## Related repositories

Changes to related repositories such as `openipc-divinus`, `openipc-builder`, `openipc-firmware`, `openipc-linux` and `u-boot-fullhan` should follow the same readable Git-flow: keep an obvious production/integration line and an obvious current development line, using short-lived topic branches only when isolation is actually useful. The no-agent-PR rule applies there as well.

Current FH8626V100 branch roles:
- reverse repository: production/state `main`; work `work/fh8626v100`. A legacy Bridge-reserved `production` ref may exist but is inactive and must not be used for project work;
- Firmware: production `master`; shared FH8626 core `work/fh8626v100`; runtime directions `work/fh8626v100-divinus` and `work/fh8626v100-majestic`;
- Builder: production `master`; single ANJIA development line `work/fh8626v100-anjia`. Divinus, Majestic and diagnostic directions are composed targets inside that one device tree, not separate active Builder branches. The former separate Majestic line is historical/archive state only; shared board fixes and runtime-fragment changes all land on the single ANJIA line;
- Linux: hardware-proven/PR-facing production line `fullhan-fh8626v100`; work `work/fh8626v100`;
- Divinus: production `master`; work `work/fh8626v100`;
- U-Boot: hardware-proven production/recovery `fh8626v100-stock-compatible`; work `fh8626v100-mainline`.

These are the current active lines, not a permanent prohibition on other branches. If a substantial task needs its own branch, create it, finish it, merge it back into the current development line after verification, and remove it when it no longer carries independent work.

This repository is the coordination authority for cross-repository work. Keep audit notes, contribution workflow, branch roles, current SHAs, evidence status and handoff state here, and synchronize those records immediately when the corresponding component state changes. Do **not** add fork-local `AGENTS.md`, audit reports or other coordination metadata to an OpenIPC component repository merely to guide later agents. Component repositories should contain only implementation and repository-owned documentation that belongs in their eventual contribution.

Before mutating any related-repository branch, determine whether the production/integration branch is the head of an open upstream pull request or otherwise acts as a submission branch. Treat it as read-only during investigation and intermediate development. Use the current development branch for ordinary work; create a topic branch only when the task is large/risky enough to benefit from isolation, then fold it back promptly after verification.

When a submitted branch needs history cleanup, reconstruct the final coherent series on the current development line or a short-lived cleanup branch from the verified upstream base, run the applicable static/build/hardware evidence gates, compare the resulting tree with the intended implementation, fold the result back into the current development line, and only then update the submission branch. If the owner has explicitly authorized a history rewrite, perform one controlled final force update rather than repeatedly force-pushing checkpoints into an active review.

Do not duplicate camera-level knowledge into those repositories. Record new camera contracts here, then implement or reference them in the repository that owns the component.

Before modifying an OpenIPC-related repository or preparing an upstream contribution, agents MUST read `docs/process/openipc-upstream-rules.md` and then re-open the relevant live upstream links listed there. The local document is a cached summary, not authority over upstream. If OpenIPC has changed repository ownership, contribution rules, review gates, branch conventions or U-Boot organization, update the local rule document before continuing.

Repository-local upstream instructions take precedence over this summary. Always inspect the current target repository's `README`, `AGENTS.md` / `CLAUDE.md`, contribution/review files and current base branch before curating a contribution.


## Execution model

- This project is operated primarily through browser/API tooling, including Koba MCP Bridge.
- Do not clone repositories, download source trees, materialize full repositories, or set up local build environments merely to inspect or validate project state.
- Do not run heavyweight kernel, firmware, Buildroot, Docker, toolchain, or image builds unless the repository owner explicitly asks for that exact operation.
- The repository owner performs the authoritative heavy builds and hardware runs. Agents prepare precise source changes, history/commit structure, build instructions when requested, and analyze the returned results.
- Prefer GitHub/API-level inspection of commits, trees, files, diffs, branches, PR state and review comments. Use lightweight static reasoning before asking for any operator-side command.
- A missing build result is a pending evidence gate, not permission to create a parallel local build environment.
