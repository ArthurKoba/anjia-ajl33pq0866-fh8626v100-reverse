# Repository map

This camera project coordinates several implementation repositories. Camera-level facts live here; component code changes live in the repository that owns the component.

| Component | Repository | Role |
|---|---|---|
| Camera project | `ArthurKoba/anjia-ajl33pq0866-fh8626v100-reverse` | Hardware/media contracts, current state, camera-specific retained source and evidence manifests |
| U-Boot | `ArthurKoba/u-boot-fullhan` | FH8626V100 open-source bootloader port and boot/recovery behavior |
| Linux | `ArthurKoba/openipc-linux` | Kernel/platform support |
| Divinus | `ArthurKoba/openipc-divinus` | Open streamer/reference implementation and FH8626 HAL/media integration |
| Firmware | `ArthurKoba/openipc-firmware` | Shared OpenIPC Buildroot external tree, SoC-family packages/runtime integration and image infrastructure |
| Builder | `ArthurKoba/openipc-builder` | Thin per-device overlay/profile and final product-image assembly for a named camera |
| Ghidra MCP infrastructure | `ArthurKoba/ghidra-mcp` | Reverse-analysis service deployment, not camera-level findings |

Current FH8626 engineering/checkpoint refs are recorded in `docs/process/upstream-integration.md`.

## Authority boundary

Do not fork camera-level knowledge into implementation repositories. When implementation or hardware validation establishes a camera contract, document it here and reference it from the component change.

Cross-repository synchronization is part of implementation completion: when a component repository changes its active SHA, branch topology, ownership split, named runtime targets, composition model or validation gates, update the current coordination documents in this repository in the same work session.

Ghidra MCP working state is not an implementation repository. Durable reverse conclusions return to this camera project; Ghidra service/deployment changes belong to `ghidra-mcp`.

## Ownership rules

Use repository ownership rather than historical file placement:

- bootloader implementation and boot/recovery features belong in `u-boot-fullhan`;
- kernel source/platform patches belong in `openipc-linux`;
- Divinus streamer/HAL/media code belongs in `openipc-divinus`;
- shared Buildroot packages, SoC-family runtime integration and generic firmware infrastructure belong in `openipc-firmware`;
- one-camera/device-specific deltas belong in `openipc-builder`;
- camera-level contracts/evidence remain here.

A preservation branch may contain code in the wrong repository. Treat that as historical integration state, not as proof of ownership.

## Current sequencing

The working sequence is foundation-first:

`U-Boot -> Linux/kernel -> Firmware ownership audit -> Divinus target closure -> Majestic product transition -> Firmware product integration -> Builder validation/promotion`.

U-Boot and kernel are already near-production/hardware-proven and should normally receive audit/reconciliation rather than broad new development. Builder source architecture is already thin and composed; its remaining role is to consume the stable lower layers and later pass build/hardware promotion gates rather than becoming a second kernel/firmware/streamer tree.

## Builder boundary

The final Builder device profile should contain only per-device deltas. For AJL33PQ0866 the current model is one shared device tree with a board-only base plus short composed Divinus/Majestic/diagnostic target fragments. Generic FH8626 defconfig/platform/kernel/filesystem selections are inherited from the selected Firmware direction rather than copied into Builder.

ANJIA-only Buildroot packages are device-local. PTZ is an optional device capability, not a mandatory boot-calibration policy. Streamer implementation remains outside Builder.

Do not leave duplicated kernel patches, copied generic FH8626 defconfigs/runtime packages or streamer implementation source in Builder after the owning repositories are prepared.

## Firmware boundary

Firmware is not a dumping ground for integration snapshots. In particular:

- kernel patches belong in Linux;
- one retail-camera profile belongs in Builder;
- streamer implementation belongs in that streamer repository;
- proprietary runtime dependencies may be packaged in Firmware only when their role, provenance, redistribution status and scope are explicit and compatible with Firmware policy.

## Branch policy

Every repository uses working branches for non-trivial changes. Verify the current upstream/base before extending an old FH8626 preservation branch. Agents do not create pull requests. Upstream-facing work must be curated from a verified current base into a coherent contribution series.
