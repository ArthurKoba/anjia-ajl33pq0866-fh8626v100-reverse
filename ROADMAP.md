# Roadmap

The project is no longer following a Builder-first integration order. The working sequence starts from the lowest platform layers, converges them on OpenIPC-native contracts, preserves already hardware-accepted recovery states, closes the Divinus target-validation gap, and only then moves to Majestic/product assembly.

## Phase 0 — foundation integration and repository sanitation

Status: **NOW**.

### 0.1 U-Boot — OpenIPC-native candidate implemented; hardware-accept, then curate

Repository: `ArthurKoba/u-boot-fullhan`.

The hardware-proven stock-compatible baseline remains on `fh8626v100-mainline`. The OpenIPC-native product candidate is implemented on `fh8626v100-openipc-native`.

The candidate already uses the standard OpenIPC 8 MiB NOR geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

with:

- `0x00000..0x0ffff` — board-specific Fullhan Boot ROM / DDR data inside the normal OpenIPC boot partition;
- `0x10000..0x3ffff` — 192 KiB U-Boot slot;
- `0x40000..0x4ffff` — OpenIPC environment;
- `0x50000` — kernel;
- `0x250000` — rootfs;
- `0x750000` — rootfs_data.

Production no longer depends on the factory Fullhan environment. OpenIPC-style boot/update variables are compiled in, while factory `kload` and vendor GPIO syntax are migration-only in the RAM target.

The candidate builds both production and RAM configurations. With useful recovery/operator features restored, production raw U-Boot is `0x2ed18` bytes and still has 4836 bytes of payload headroom in the 192 KiB slot.

Current work is now to:

1. build and measure the final intended FH8626 OpenIPC `uImage` and target the standard 2 MiB partition;
2. reduce kernel config/compression/built-in footprint first if it does not fit, before accepting any special layout;
3. perform the documented native full-layout migration with external SPI recovery available;
4. cold-boot and verify ROM -> U-Boot -> OpenIPC kernel/rootfs/environment/network on the physical AJL33PQ0866;
5. only after that hardware gate, promote the candidate and curate/squash the U-Boot contribution series;
6. update the stale repository workflow to native artifact names using an identity with GitHub workflow-write permission;
7. run current U-Boot contribution/style/checkpatch gates;
8. present OpenIPC maintainers with source, hardware evidence, Boot-ROM provenance and artifact scheme and obtain the intended OpenIPC U-Boot repository ownership.

Do not replace the hardware-proven baseline before native cold boot passes, and do not advertise the board-specific artifact as universal FH8626V100.

Detailed findings are in `docs/hardware/uboot-port.md`.

### 0.2 Linux/kernel — audit, exact upstream reconciliation, and 2 MiB gate

Repository: `ArthurKoba/openipc-linux`.

FH8626V100 kernel/platform support is already hardware-proven across the exercised board platform. The default action is review and reconciliation rather than feature development.

Current work is to:

- verify the current FH8626V100 branch against accepted platform contracts;
- locate and verify the exact upstream pull request and reconcile comments/drift;
- audit configuration and built-in/module choices against OpenIPC conventions;
- explicitly classify retained proprietary Fullhan media modules separately from open kernel/platform support;
- build and measure the final target `uImage` because it is the immediate U-Boot native-layout gate;
- change kernel code only for a defect, upstream review issue, regression or justified config/size cleanup.

### 0.3 Firmware — ownership and placement audit

Repository: `ArthurKoba/openipc-firmware`.

The current FH8626V100 firmware branch is a preservation snapshot, not an accepted final architecture. It mixes several ownership domains and must be classified before cleanup.

Routing rule:

- kernel source/patches -> `openipc-linux`;
- one-camera/device deltas -> `openipc-builder`;
- Divinus implementation -> `openipc-divinus`;
- camera-level hardware/media contracts -> this repository;
- genuinely shared SoC-family runtime packages, load scripts and acceptable vendor runtime dependencies -> `openipc-firmware`.

The historical FH8626 `3 MiB kernel / rootfs at 0x450000` rule is preservation state, not product architecture. If the final kernel fits the normal 2 MiB OpenIPC partition, Firmware must use the standard image boundaries rather than carrying that special case.

Do not delete or move material merely because its present location is wrong. First identify owner, provenance and active dependency, then curate it into the owning repository.

## Phase 1 — Divinus completion and hardware acceptance

Repository: `ArthurKoba/openipc-divinus`.

Divinus is the current open reference implementation. The latest FH8626V100 source contains the newest native-HAL/media work but the final migration state has not yet been validated on the physical camera.

Required sequence:

1. establish the exact latest source/build candidate and build it reproducibly;
2. deploy that candidate to the camera without confusing it with an older listener/process;
3. validate sensor/media initialization and visible image output;
4. validate VENC and sustained RTSP including restart/reconnect behavior;
5. validate WIDE/TELE switching and required board sensor bootstrap behavior;
6. validate ISP/exposure/color/day-night behavior at the evidence level actually exercised;
7. validate microphone, speaker/two-way audio where the candidate claims support;
8. validate integrations Divinus owns or exposes without duplicating board policy into the streamer;
9. repair only failures reproduced on the latest candidate;
10. once accepted, curate a clean upstream-ready FH8626V100 Divinus contribution series.

The present Divinus WIP must not be called hardware-accepted until this pass is complete.

## Phase 2 — Majestic product transition

Majestic remains the intended product streamer after the Divinus reference path is closed.

Reuse architecture-neutral camera contracts and low-level knowledge proven during Divinus work, but do not assume source can be copied mechanically into Majestic.

Validation order:

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

After platform and streamer ownership boundaries are clear, keep only shared and reusable runtime integration in `openipc-firmware`.

This includes SoC-family packaging/load policy and dependencies that truly belong to Firmware. Camera-specific behavior and streamer source remain in their owning repositories.

## Phase 4 — Builder final device profile

Repository: `ArthurKoba/openipc-builder`.

Builder is deliberately last. It is the thin per-device assembly layer and must not become a second firmware/kernel/streamer repository.

The final AJL33PQ0866 Builder profile may contain only physical-device deltas such as:

- device defconfig/package selection;
- first-boot/device-specific GPIO/bootstrap policy;
- sensor/lens selection defaults and device runtime configuration;
- audio/PTZ/illumination configuration where the owning service consumes it;
- excludes/size policy and other device-only packaging data.

Do not retain duplicate kernel patches, Divinus/Majestic implementation source, generic FH8626 runtime code or broad media HAL code in Builder once their owners are ready.

The preserved pre-Majestic Builder checkpoint is only a reference baseline. The later Majestic commit stays isolated until Phase 2.

## Upstream curation

Nothing in preservation/WIP or hardware-test candidate branches is automatically a final upstream contribution set.

For each repository:

- verify current upstream/base first;
- preserve hardware-proven behavior;
- remove generated/debug/duplicated material from the contribution set;
- separate unrelated responsibilities;
- rebuild coherent commits on a working branch;
- test at the evidence level appropriate to that repository;
- leave final pull-request creation to the repository owner.

## Reverse boundary

Broad reverse is not a roadmap phase. Use canonical Ghidra MCP only for a concrete implementation blocker, contradictory target evidence or a narrowly scoped missing contract.
