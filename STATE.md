# Current project state

Status: `ACTIVE / UBOOT_OPENIPC_NATIVE_CANDIDATE / DIVINUS_REFERENCE / MAJESTIC_TARGET`.

Checked: `2026-09-17`.

## Authority

- GitHub owns current camera-level source, documentation, contracts, state and evidence manifests.
- Google Drive owns heavy or unique primary evidence referenced by SHA-256 from `evidence/MANIFEST.tsv`.
- Ghidra MCP through Koba MCP Bridge is the canonical mutable reverse-analysis workspace.

## Target hardware

- SoC: FH8626V100.
- Board/camera: ANJIA AJL33PQ0866.
- Sensors: dual GC1054 MIPI.
- Exercised native mode: 1280x720 @ 25 fps.
- WIDE is the product default lens.
- GPIO5 cold-boot sequencing is required for TELE visibility before media startup.
- Stock lens selection uses GPIO4/GPIO14 and coordinates switching with the media pipeline.
- `/dev/fh_pwm` PTZ control is hardware-proven on the exercised board after correcting swapped motor connectors.

## Current repository checkpoints

These refs are working-state locators, not automatic upstream bases or contribution sets.

### U-Boot

Repository: `ArthurKoba/u-boot-fullhan`.

Hardware-proven reference:

- branch: `fh8626v100-mainline`
- sanitized observed tip: `49fe46e9ddb786e232d1359f9cee68c914a3a8db`
- role: stock-compatible recovery/reference baseline

OpenIPC-native candidate:

- branch: `fh8626v100-openipc-native`
- observed tip: `99c477674acc250b3177e3ae6eaaa1680c28c809`
- relation: five commits ahead of `fh8626v100-mainline`
- evidence level: `SOURCE/BUILD CANDIDATE`, not `HARDWARE_PASS`

The OpenIPC-native candidate is now implemented rather than merely planned. It changes the production contract to the standard OpenIPC 8 MiB NOR geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.

For this board the standard 256 KiB `boot` partition contains a Fullhan-specific internal split only: 64 KiB reconstructed Boot ROM data at `0x00000` and a fixed 192 KiB U-Boot slot at `0x10000`. Environment is at the standard OpenIPC `0x40000`, kernel at `0x50000`, rootfs at `0x250000`, and rootfs_data at `0x750000`.

The candidate no longer depends on the Fullhan factory environment in production. `kload`, factory GPIO syntax and other stock compatibility are opt-in only in the RAM migration/recovery target. The production environment now uses OpenIPC-style `mtdpartsnor8m`, `setnor8m`, `bootcmdnor`, `uknor8m`, `urnor8m`, `uImage.${soc}` and `rootfs.squashfs.${soc}` contracts and mounts rootfs from `/dev/mtdblock3`.

The board identity is explicit: FH8626V100 support is reusable, while the validated Boot-ROM/DDR data, RMII wiring and persistent-flash migration contract belong to ANJIA AJL33PQ0866 until another FH8626 board is independently proven.

The candidate build compiles both production and RAM targets. With TFTP upload, TFTP tuning variables, line editing, autocomplete, long help and `sleep` restored, the production raw U-Boot is `0x2ed18` bytes. The 192 KiB payload slot reserves four bytes for the ROM checksum fixup and still has 4836 bytes of free payload budget.

The native packer emits a board-specific 256 KiB OpenIPC `boot` artifact plus 64 KiB bootstrap, 192 KiB padded U-Boot, raw production binary, RAM recovery binary and SHA-256 manifest. It validates the relocated U-Boot descriptor (`flash=0x10000`, `size=0x30000`, load/entry `0xa0800000`) while retaining the recovered board Boot-ROM data.

The existing repository workflow still contains stock-artifact post-build checks. The GitHub App available to this project is not permitted to update `.github/workflows/build.yml`, so the auto-run currently compiles both native targets and generates the native artifacts successfully, then reports failure only when the stale workflow looks for the removed stock artifact names. Do not interpret that final workflow status as a compile failure. The workflow itself must be updated by an identity with GitHub workflow-write permission before contribution.

The remaining hard gates are:

1. build and measure the final intended FH8626 OpenIPC `uImage`; it must fit the standard 2 MiB kernel partition or the kernel configuration must be reduced before considering a custom layout;
2. perform the documented one-time migration with external SPI recovery available;
3. cold-boot the complete OpenIPC-native layout on the physical AJL33PQ0866;
4. verify MTD/environment/kernel/rootfs/network behavior after that cold boot.

Until those gates pass, keep `fh8626v100-mainline` as the proven recovery baseline and do not replace it with the native candidate.

Detailed findings: `docs/hardware/uboot-port.md`.

### Linux/kernel

Repository: `ArthurKoba/openipc-linux`.

- branch: `fullhan-fh8626v100`
- observed tip: `ebf5d776c748edbd58c1aaf8be9d5b2639a16834`
- current FH8626V100 series: two commits over the `fullhan-fh8852v200` lineage

The exercised platform support is hardware-proven across boot, Ethernet, storage, watchdog, GPIO/pinmux, PWM, USB and the board paths documented in this repository. The operator reports that FH8626V100 kernel support has already been submitted upstream; the exact upstream PR/status must be independently verified before deciding whether further kernel work is required.

The immediate kernel task is also the final OpenIPC layout gate: build the intended OpenIPC `uImage`, measure it exactly, and make the standard 2 MiB partition the default target. A historical 3 MiB reservation is not a reason to retain a custom partition map.

### Divinus

Repository: `ArthurKoba/openipc-divinus`.

- branch: `fh8626v100-canonical`
- observed tip: `1e624bd5aca97ba772413d2b00a10314d1db039f`
- base integration commit: `8d400262898e8e82df6171fde7e8911ec7930249` (`Add generic FH8626V100 platform support`)
- top preservation commit: `WIP: preserve FH8626V100 native HAL migration state`

The tip contains the newest native-HAL/media/ISP/audio/transport migration work. That final WIP state has not yet completed physical-camera acceptance. The next Divinus task is to build the exact latest candidate, deploy it to the camera, validate it end-to-end, and repair only reproduced failures before curating an upstream-ready series.

### Firmware

Repository: `ArthurKoba/openipc-firmware`.

- branch: `fh8626v100-platform`
- observed tip: `6db66c53971fda8ba733f370a965e52cd53fb61b`
- relation to its preserved base: one WIP commit
- top commit: `WIP: preserve FH8626V100 platform integration state`

This branch is a preservation snapshot, not an accepted repository layout. The snapshot currently mixes kernel patches, board-specific support, a large Divinus patch, proprietary Fullhan modules/libraries, camera media-owner/ISP source and host tests. Every retained item must be classified by ownership before cleanup or upstream preparation.

Its historical `3 MiB kernel / 1 MiB rootfs_data / rootfs at 0x450000` arrangement is preservation evidence only. If the final curated FH8626 kernel fits the standard 2 MiB OpenIPC partition, Firmware must use the normal OpenIPC 8 MiB assembly boundaries instead of carrying that old special layout.

### Builder

Repository: `ArthurKoba/openipc-builder`.

- branch: `fh8626v100-anjia-ajl33pq0866`
- observed tip: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
- stable pre-Majestic checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`
- Divinus/device-profile predecessor: `c7577cb3f70b4531e9ec9e686c5275bf5f170c3d`
- Majestic experiment: the single commit `bcf8658e...` on top of `5603a701...`

The pre-Majestic checkpoint is the reference device-integration baseline while Divinus is completed. The later Majestic commit is an isolated experiment and must not silently become the Builder baseline. Builder itself is currently behind/diverged from newer upstream `master`, so later Builder work must begin from the current upstream base.

## Ownership boundary

The current OpenIPC repository rules reinforce the intended split:

- kernel source and kernel patches belong in `openipc-linux`;
- support specific to one retail camera belongs in `openipc-builder`;
- Divinus implementation belongs in `openipc-divinus`;
- genuinely shared firmware packages, SoC-family drivers/load scripts and rootfs integration belong in `openipc-firmware`;
- camera-level contracts and evidence remain here.

Existing WIP placement is historical evidence, not proof of correct ownership.

## Sensor / ISP / media

Durable camera contracts are documented under `docs/sensor/`, `docs/isp/`, `docs/media/` and `docs/architecture/`.

Broad reverse of the exercised stock path is not an active objective. New reverse work should answer a concrete implementation or validation question in Ghidra MCP and then promote the durable result into current Git documentation or reusable source.

Static/source/reverse coverage is not equivalent to target runtime acceptance. Hardware acceptance remains explicitly labeled.

## Divinus

Divinus is the open reference path that must be closed before the Majestic product transition. Existing FH8626 source is substantial, but the latest native migration must still be tested on the physical camera before it can be called accepted.

## Majestic

Majestic remains the intended product path after the Divinus reference implementation is closed.

An FH8852-family Majestic experiment exists in the preserved Builder branch and process startup has been observed historically, but that is not a current product baseline. Majestic acceptance still requires a pinned reproducible candidate followed by VI -> VENC -> sustained RTSP -> ISP -> audio/control validation.

## Audio

RTX microphone and speaker paths are hardware-proven on this board. Product-streamer integration remains to be validated in the selected final path.

## PTZ and illumination

The accepted low-level PTZ backend is `/dev/fh_pwm`. Illumination/IR-cut board contracts are already documented. Remaining work is service/streamer/product integration and calibration, not rediscovery of the electrical backend.

## Curated source

Reusable camera-specific engineering components are retained under `source/fh8626v100/components/`. They are implementation references/contracts, not permission to duplicate the same source into Firmware, Builder and Divinus.

## Evidence

The retained GC1054 vendor sensor binary is external evidence:

- SHA-256: `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`
- role/locator: `evidence/MANIFEST.tsv`

Firmware, dumps and other heavy primary evidence remain external and SHA-addressed.

## Immediate engineering sequence

1. build/measure the final FH8626 OpenIPC kernel against the 2 MiB standard partition;
2. if it fits, hardware-test the complete `fh8626v100-openipc-native` U-Boot/layout migration and retain standard OpenIPC geometry;
3. after hardware acceptance, curate/squash the native U-Boot contribution series and resolve the workflow update/target OpenIPC repository ownership;
4. reconcile the kernel with its exact upstream PR state;
5. classify the mixed Firmware preservation snapshot by repository ownership;
6. build and target-test the latest Divinus candidate until its FH8626 path is complete;
7. transition the product path to Majestic;
8. integrate shared runtime pieces into Firmware;
9. rebuild the final thin Builder device profile last.
