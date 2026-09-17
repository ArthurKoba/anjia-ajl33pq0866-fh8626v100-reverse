# Roadmap

The project is no longer following a Builder-first integration order. The working sequence starts from the lowest finished platform layers, preserves already hardware-accepted work, closes the current Divinus target-validation gap, and only then moves to Majestic/product assembly.

## Phase 0 — foundation audit and repository sanitation

Status: **NOW**.

### 0.1 U-Boot — audit, document, freeze

Repository: `ArthurKoba/u-boot-fullhan`.

The FH8626V100 port is already a working modern open-source U-Boot implementation and is used on the target camera. Treat it as near-production, not as a new porting project.

Current work is to:

- verify the current FH8626V100 branch against the actual operational needs of OpenIPC;
- distinguish required boot/recovery/update features from optional vendor/development commands;
- audit the port against current U-Boot/OpenIPC layout and contribution conventions;
- document intentionally omitted functionality when it is not required;
- change working code only for a concrete defect, missing required capability or contribution-quality issue.

A future OpenIPC-owned general U-Boot fork/repository strategy is a separate long-term organizational question and is not a blocker for this camera.

### 0.2 Linux/kernel — audit and upstream reconciliation

Repository: `ArthurKoba/openipc-linux`.

FH8626V100 kernel/platform support is already hardware-proven across the exercised board platform. The default action is therefore review and reconciliation rather than further feature development.

Current work is to:

- verify the current FH8626V100 branch against the accepted camera/platform contracts;
- verify the exact upstream pull-request state and reconcile any review comments or drift;
- audit configuration, built-in/module choices and OpenIPC conventions;
- explicitly classify retained proprietary Fullhan media modules separately from open kernel/platform support;
- change kernel code only when a defect, upstream review issue or real platform regression requires it.

### 0.3 Firmware — ownership and placement audit

Repository: `ArthurKoba/openipc-firmware`.

The current FH8626V100 firmware branch is a preservation snapshot, not an accepted final architecture. It currently mixes several ownership domains and must be classified before any cleanup is performed.

Routing rule:

- kernel source/patches -> `openipc-linux`;
- one-camera/device deltas -> `openipc-builder`;
- Divinus implementation -> `openipc-divinus`;
- camera-level hardware/media contracts -> this repository;
- genuinely shared SoC-family runtime packages, load scripts and legally/provenance-acceptable vendor runtime dependencies -> `openipc-firmware`.

Do not delete or move material merely because its present location is wrong. First identify its current owner, provenance and active dependency, then curate it into the repository that owns it.

## Phase 1 — Divinus completion and hardware acceptance

Repository: `ArthurKoba/openipc-divinus`.

Divinus is the current open reference implementation. The latest FH8626V100 source contains the newest native-HAL/media work but the final migration state has not yet been validated on the physical camera.

Required sequence:

1. establish the exact latest source/build candidate and build it reproducibly;
2. deploy that candidate to the camera without confusing it with an older listener/process;
3. validate sensor/media initialization and visible image output;
4. validate VENC and sustained RTSP including restart/reconnect behavior;
5. validate WIDE/TELE switching and the required board sensor bootstrap boundary;
6. validate ISP/exposure/color/day-night behavior at the evidence level actually exercised;
7. validate microphone, speaker/two-way audio where the candidate claims support;
8. validate the integrations Divinus owns or exposes for the camera without duplicating board policy into the streamer;
9. repair only failures reproduced on the latest candidate;
10. once the target path is accepted, curate a clean upstream-ready FH8626V100 Divinus contribution series.

The present Divinus WIP must not be called hardware-accepted until this pass is complete.

## Phase 2 — Majestic product transition

Majestic remains the intended product streamer after the Divinus reference path is closed.

Reuse architecture-neutral camera contracts and low-level knowledge proven during the Divinus work, but do not assume Divinus source can be copied mechanically into Majestic. The two streamers have different integration and ownership surfaces.

Validation order remains:

1. pin exact Majestic binary/build provenance, runtime libraries and configuration;
2. VI;
3. VENC;
4. sustained RTSP;
5. ISP/color/exposure/day-night;
6. audio capture/playback/two-way behavior;
7. PTZ/control integration;
8. illumination/IR-cut integration;
9. restart/reconnect/regression behavior.

The existing Builder Majestic experiment is reference material for this phase, not the current baseline.

## Phase 3 — firmware product integration

After the platform and streamer ownership boundaries are clear, keep only shared and reusable runtime integration in `openipc-firmware`.

This includes SoC-family packaging/load policy and dependencies that truly belong to the firmware tree. Camera-specific behavior and streamer source remain in their owning repositories.

## Phase 4 — Builder final device profile

Repository: `ArthurKoba/openipc-builder`.

Builder is deliberately last. It is the thin per-device assembly layer and must not become a second firmware/kernel/streamer repository.

The final AJL33PQ0866 Builder profile may contain only the deltas required to assemble this physical camera, such as:

- device defconfig/package selection;
- first-boot/device-specific GPIO and bootstrap policy;
- sensor/lens selection defaults and camera-specific runtime configuration;
- audio/PTZ/illumination device configuration where the owning service consumes it;
- excludes/size policy and other device-only packaging data.

Do not keep duplicate kernel patches, Divinus/Majestic implementation source, generic FH8626 runtime libraries or broad media HAL code in Builder once their owning repositories are ready.

The preserved pre-Majestic Builder checkpoint is the reference baseline for Divinus-era device integration. The later Majestic experiment stays isolated until Phase 2.

## Upstream curation

Nothing in the current preservation/WIP branches is automatically an upstream contribution set.

For each repository:

- verify its current upstream/base first;
- preserve hardware-proven behavior;
- remove generated/debug/duplicated material from the contribution set;
- separate unrelated responsibilities;
- rebuild coherent commits on a working branch;
- test at the evidence level appropriate to that repository;
- leave pull-request creation to the repository owner.

## Reverse boundary

Broad reverse is not a roadmap phase. Use the canonical Ghidra MCP project only for a concrete implementation blocker, contradictory target evidence or a narrowly scoped missing contract.
