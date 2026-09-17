# Roadmap

The project is no longer following a Builder-first integration order. The working sequence starts from the lowest finished platform layers, preserves already hardware-accepted work, closes the current Divinus target-validation gap, and only then moves to Majestic/product assembly.

## Phase 0 — foundation audit and repository sanitation

Status: **NOW**.

### 0.1 U-Boot — migrate from stock-compatible layout to OpenIPC-native layout, then curate

Repository: `ArthurKoba/u-boot-fullhan`.

The FH8626V100 port is a working modern open-source U-Boot implementation already used on the target camera. The 2026-09-17 source/architecture audit found no missing capability required by the already exercised boot/recovery path.

The currently proven partitioning is a stock-compatible migration/reference state, not the final OpenIPC product layout.

The target is to converge on the current OpenIPC 8 MiB NOR geometry where technically possible:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

FH8626 can potentially match the first 320 KiB exactly by using:

- `0x00000..0x0ffff` — Fullhan Boot ROM container / DDR parameters;
- `0x10000..0x3ffff` — 192 KiB U-Boot slot;
- `0x40000..0x4ffff` — OpenIPC environment;
- `0x50000` — kernel start, already matching OpenIPC.

Current work is to:

- build and measure the final intended FH8626 OpenIPC kernel image;
- use the standard OpenIPC 2 MiB kernel / `0x250000` rootfs boundary if the final kernel fits;
- if it does not fit, first audit kernel config/compression/size before accepting a special partition map;
- change the reconstructed Fullhan U-Boot descriptor from flash offset `0x20000` to `0x10000` and move environment storage to `0x40000` for the OpenIPC-native target;
- keep the already-proven stock-compatible target/state as a recovery and migration reference rather than forcing its partitioning into the product design;
- hardware-test the new descriptor/layout on a recoverable board before calling it accepted;
- make the ANJIA AJL33PQ0866 board-specific boundary explicit instead of presenting the artifact as universal FH8626V100;
- fold the final `ethaddr` WIP fix into a clean logical commit;
- keep generic DesignWare prerequisites separate/reviewable from Fullhan-specific quirks;
- document the Boot ROM container reconstruction provenance clearly;
- run current U-Boot contribution/style/checkpatch gates over the curated series;
- ask OpenIPC maintainers which U-Boot repository ownership they want before publishing organization-level source/artifacts.

Detailed findings and migration design are in `docs/hardware/uboot-port.md`.

### 0.2 Linux/kernel — audit and upstream reconciliation

Repository: `ArthurKoba/openipc-linux`.

FH8626V100 kernel/platform support is already hardware-proven across the exercised board platform. The default action is review and reconciliation rather than feature development.

Current work is to:

- verify the current FH8626V100 branch against the accepted camera/platform contracts;
- verify the exact upstream pull-request state and reconcile any review comments or drift;
- audit configuration, built-in/module choices and OpenIPC conventions;
- explicitly classify retained proprietary Fullhan media modules separately from open kernel/platform support;
- measure the final target `uImage` because it determines whether standard OpenIPC 8 MiB partitioning is viable;
- change kernel code only when a defect, upstream review issue, regression or justified config/size cleanup requires it.

### 0.3 Firmware — ownership and placement audit

Repository: `ArthurKoba/openipc-firmware`.

The current FH8626V100 firmware branch is a preservation snapshot, not an accepted final architecture. It currently mixes several ownership domains and must be classified before cleanup.

Routing rule:

- kernel source/patches -> `openipc-linux`;
- one-camera/device deltas -> `openipc-builder`;
- Divinus implementation -> `openipc-divinus`;
- camera-level hardware/media contracts -> this repository;
- genuinely shared SoC-family runtime packages, load scripts and legally/provenance-acceptable vendor runtime dependencies -> `openipc-firmware`.

The historical FH8626 `3 MiB kernel / rootfs at 0x450000` rule is not a permanent requirement. If the final kernel fits the normal 2 MiB OpenIPC partition, drop the special layout and use the standard image assembly path.

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
