# Current project state

Status: `ACTIVE / FOUNDATION_AUDIT / DIVINUS_REFERENCE / MAJESTIC_TARGET`.

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

- branch: `fh8626v100-mainline`
- observed tip: `ae63365e10b38e5b9ed3bd5173a0ed3e5c8f9996`
- relation to repository `main`: five FH8626V100 commits ahead
- top preservation commit: `WIP: preserve latest FH8626V100 U-Boot state`

The port is already hardware-used and near-production. It is based on modern open-source U-Boot rather than copied vendor U-Boot code. Current work is an OpenIPC/U-Boot feature and contribution-quality audit, not a new port.

### Linux/kernel

Repository: `ArthurKoba/openipc-linux`.

- branch: `fullhan-fh8626v100`
- observed tip: `ebf5d776c748edbd58c1aaf8be9d5b2639a16834`
- current FH8626V100 series: two commits over the `fullhan-fh8852v200` lineage

The exercised platform support is hardware-proven across boot, Ethernet, storage, watchdog, GPIO/pinmux, PWM, USB and the board paths documented in this repository. The operator reports that FH8626V100 kernel support has already been submitted upstream; the exact upstream PR/status must be independently verified before deciding whether any further kernel work is required.

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

### Builder

Repository: `ArthurKoba/openipc-builder`.

- branch: `fh8626v100-anjia-ajl33pq0866`
- observed tip: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
- stable pre-Majestic checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`
- Divinus/device-profile predecessor: `c7577cb3f70b4531e9ec9e686c5275bf5f170c3d`
- Majestic experiment: the single commit `bcf8658e...` on top of `5603a701...`

The pre-Majestic checkpoint is the reference device-integration baseline while Divinus is completed. The later Majestic commit is an isolated experiment and must not silently become the Builder baseline. Builder itself is currently behind/diverged from newer upstream `master`, so any later Builder work must begin by verifying the current upstream base rather than extending the old branch in place.

## Ownership boundary discovered during reconciliation

The current OpenIPC repository rules reinforce the intended split:

- kernel source and kernel patches belong in `openipc-linux`;
- support specific to one retail camera belongs in `openipc-builder`;
- Divinus implementation belongs in `openipc-divinus`;
- genuinely shared firmware packages, SoC-family drivers/load scripts and rootfs integration belong in `openipc-firmware`;
- camera-level contracts and evidence remain here.

Existing WIP placement is evidence of historical integration work, not proof of correct ownership.

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

Reusable camera-specific engineering components are retained under:

`source/fh8626v100/components/`

They are implementation references and contracts, not a license to duplicate the same source into Firmware, Builder and Divinus. Production code belongs in the repository that owns the component.

## Evidence

The retained GC1054 vendor sensor binary is external evidence:

- SHA-256: `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`
- role/locator: `evidence/MANIFEST.tsv`

Firmware, dumps and other heavy primary evidence remain external and SHA-addressed.

## Immediate engineering sequence

1. audit/freeze U-Boot;
2. audit/reconcile kernel and exact upstream PR state;
3. classify the mixed Firmware preservation snapshot by repository ownership;
4. build and target-test the latest Divinus candidate until its FH8626 path is complete;
5. transition the product path to Majestic;
6. integrate shared runtime pieces into Firmware;
7. rebuild the final thin Builder device profile last.
