# Current project state

Status: `ACTIVE / UBOOT_OPENIPC_NATIVE_SOURCE_READY / KERNEL_GATE / FIRMWARE_CLEAN_CANDIDATE / DIVINUS_REFERENCE / MAJESTIC_TARGET`.

Checked: `2026-09-18`.

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

FH8626 branch state is kept intentionally readable: one shared core/development line carries platform work, while substantial runtime directions may have their own long-lived branches and are periodically reconciled with that core. Short-lived topic branches should be folded back and removed after verification. Historical states are tags/SHAs, not active branches. In this reverse repository, `main` is production/state and `work/fh8626v100` is the current development line. The old `production@c1e94ad41cf3ff7a9b9f862e0526e5425191df77` ref is Bridge-reserved, inactive and must be ignored.

### U-Boot

Repository: `ArthurKoba/u-boot-fullhan`.

Current OpenIPC-native working line:

- branch: `fh8626v100-mainline`
- observed tip: `7ac0aa7e83fb859b90c8b5e11bf617e367f2bd1d`
- evidence level: `SOURCE/BUILD ACCEPTED`, not `HARDWARE_PASS`

Preserved hardware-proven stock-compatible reference:

- branch: `fh8626v100-stock-compatible`
- tip: `49fe46e9ddb786e232d1359f9cee68c914a3a8db`
- role: recovery/evidence baseline for the already exercised factory-compatible boot geometry

The agreed OpenIPC-native changes are now in the actual U-Boot implementation, not only in roadmap/documentation. `fh8626v100-mainline` was advanced by normal fast-forward after preserving the old hardware-proven state on `fh8626v100-stock-compatible`.

The production contract is the normal OpenIPC 8 MiB NOR geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.

For AJL33PQ0866 the standard 256 KiB `boot` partition contains the Fullhan-specific internal split only: 64 KiB reconstructed Boot-ROM/DDR data at `0x00000` and a 192 KiB physical U-Boot slot at `0x10000`. Environment is at `0x40000`, kernel at `0x50000`, rootfs at `0x250000`, and rootfs_data begins at `0x750000`.

Production no longer depends on the factory Fullhan environment. `kload`, factory `gpio <pin> out <0|1>` syntax and other factory compatibility are opt-in only in the RAM migration/recovery target through `CONFIG_FH8626V100_STOCK_COMPAT`.

The production environment now follows the normal OpenIPC NOR naming/semantics rather than carrying FH8626-only updater conventions. It exposes `kernaddr`, `kernsize`, `rootaddr`, `rootsize`, `mtdpartsnor8m`, `setnor8m`, `cmdnor`, `bootcmdnor`, `updatetool`, `ubnor`/`ubwrite`, `uknor`/`ukwrite`, and `urnor`/`urwrite`. Kernel and rootfs use `uImage.${soc}` and `rootfs.squashfs.${soc}`. The U-Boot artifact is deliberately board-qualified as `u-boot-${soc}-${board}-nor.bin` because the recovered Boot-ROM/DDR data is currently proven only for AJL33PQ0866.

The native release packer now emits two useful upper-level artifacts:

- `...-boot.bin`: exact 256 KiB OpenIPC `boot` partition;
- `...-nor.bin`: 320 KiB OpenIPC updater image containing the 256 KiB boot partition followed by an erased 64 KiB environment sector, matching the normal `ubwrite` boundary at kernel offset `0x50000`.

The native ROM descriptor is generated from the actual linked U-Boot rather than preserving factory descriptor magic. For the verified build at `227bcb40...`, `build.sh` reported:

- raw U-Boot: `0x2ef20`;
- aligned ROM payload: `0x2f000`;
- calculated JAMCRC: `0x0c6d419b`;
- physical U-Boot slot: `0x30000`;
- boot partition: `0x40000`;
- NOR updater image: `0x50000`.

The descriptor points to flash offset `0x10000`, retains load/entry `0xa0800000`, uses actual raw size, `0x100`-aligned payload size and the calculated JAMCRC. The remaining bytes in the 192 KiB physical slot stay erased. The historical fixed stock descriptor size/JAMCRC remain only in the stock parser/evidence path.

`build.sh` is now self-validating: it runs the native packer unit tests, builds production U-Boot, generates and inspects the native NOR artifact, then builds the RAM recovery target. The observed build ran 8 native artifact tests successfully, compiled both targets, and the generated artifact inspector accepted the complete native layout/descriptor.

The repository's existing GitHub workflow still has a stale post-build step that looks for the removed stock artifact names and stock descriptor. The GitHub App available to the project has no `workflows` write permission, so `.github/workflows/build.yml` could not be updated through the permitted Bridge path. The workflow therefore ends red after the successful build/self-validation stages. This is an infrastructure/documentation debt, not an implementation compile failure; it must be corrected later by an identity with workflow-write permission. No workaround or fake compatibility artifacts should be added merely to make the stale check green.

The native state still is **not hardware acceptance**. The remaining gates are:

1. build and measure the final intended FH8626 OpenIPC `uImage`; it must fit the standard 2 MiB kernel partition or kernel configuration/compression/built-in choices must be reviewed before considering any custom layout;
2. perform the documented one-time migration with verified full-flash backup and external SPI recovery available;
3. cold-boot the complete OpenIPC-native layout on the physical AJL33PQ0866;
4. verify Boot-ROM -> U-Boot relocation, environment at `0x40000`, MTD map, kernel/rootfs, Ethernet/MAC propagation and normal OpenIPC update variables after the cold boot.

Contribution history is not final yet. The native branch accumulated iterative implementation commits and should be rebuilt/squashed into a clean upstream-facing series only after hardware acceptance, so contribution curation does not obscure the exact code that was tested on the camera.

Detailed findings: `docs/hardware/uboot-port.md`.

### Linux/kernel

Repository: `ArthurKoba/openipc-linux`.

- PR-facing/integration branch: `fullhan-fh8626v100`
- live observed tip: `0dfafa643770d78389e444c03f46f1711662eda6`
- verified parent lineage: `fullhan-fh8852v200@ee1ef844294bfa1ff15b2f0522d35c987a16a220`
- isolated OpenIPC MTD fix: `archive/fh8626v100-mtd-fix-20260918@28a923a9d9598a9a4e2c6c0ee4b2eee26698731e`
- exploratory reconstruction: `archive/fh8626v100-clean-series-20260918@868bdd8ddde7a35c2c744e5706941d5e1f9faadf`
- curated staging series: `work/fh8626v100@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`

The former `ebf5d776c748edbd58c1aaf8be9d5b2639a16834` locator is stale after branch identity rewrite/repoint and is not the current integration tip.

The exercised historical platform remains `HARDWARE_PASS`. Its accepted `uImage` was 1,583,456 bytes, so the tested tree fit the standard 2 MiB kernel partition.

The curated staging series is `SOURCE_CONFIRMED / AUDIT`, not new hardware acceptance. It fixes the OpenIPC MTD contract, restores the required Fullhan boardconfig input, removes retail naming from the SD0 Kconfig option, wires the already-existing FH8626 PWM v2 Kconfig selection to `pwmv2.o`, and adds the missing non-DT AXI-DMA platform registration. Generic fixes are separated from SoC enablement while hardware-tested RMII behavior is preserved.

The PR-facing branch is intentionally untouched. The no-fixup 13-commit staging series has now been rebuilt from the verified base and is ready for the owner's authoritative build/check/hardware gates before any single submission-branch history update.

As rechecked on 2026-09-18, the public OpenIPC/linux branch list still does not show FH8626V100. The operator reports an upstream submission exists, but the exact PR number/status has not been independently verified through the currently accessible API.

Detailed audit: `docs/process/fh8626-kernel-series-audit.md`.

### Divinus

Repository: `ArthurKoba/openipc-divinus`.

- branch: `work/fh8626v100`
- observed tip: `1e624bd5aca97ba772413d2b00a10314d1db039f`
- base integration commit: `8d400262898e8e82df6171fde7e8911ec7930249` (`Add generic FH8626V100 platform support`)
- top preservation commit: `WIP: preserve FH8626V100 native HAL migration state`

The tip contains the newest native-HAL/media/ISP/audio/transport migration work. That final WIP state has not yet completed physical-camera acceptance. The next Divinus task is to build the exact latest candidate, deploy it to the camera, validate it end-to-end, and repair only reproduced failures before curating an upstream-ready series.

### Firmware

Repository: `ArthurKoba/openipc-firmware`.

Preservation snapshot:

- tag: `archive/fh8626v100-platform-20260918`
- tip: `f4bf49da6ef355c9e733e00d774efe403513b1d4`
- role: historical mixed WIP/evidence only

Clean integration candidate:

- branch: `work/fh8626v100`
- tip: `eabd1ccd4684af6997771269c4655f7e4435bcec`
- pre-config-audit checkpoint: `f9146dd42a2f606d305ebccd301268848de26880`
- base: `master@47ccdbee45fa5b8eee69c25c7af656cd5d35a28e`
- diff: only the FH8626 generic kernel config, generic lite defconfig and CI registration

The clean/core branch consumes `openipc-linux/work/fh8626v100@357c2d13...` directly, carries no FH8626 kernel patch directory, and is now streamer-neutral: it deliberately selects neither Divinus nor Majestic. It also carries no AJL board package/fragment, no factory `.ko/.so/.bin`, and no local/patch copy of Divinus. The temporary exact Linux tarball points at the ArthurKoba fork only until the curated series lands in `OpenIPC/linux`; that pin is not upstream-ready provenance.

Firmware now inherits the standard OpenIPC 8 MiB assembly budget: 2 MiB kernel plus 5 MiB SquashFS, while the Linux source supplies `256K boot + 64K env + 2048K kernel + 5120K rootfs + rest rootfs_data` (704 KiB remainder on 8 MiB NOR). The prior 3 MiB-kernel arrangement remains preservation evidence only.

Runtime direction branches are layered on that core:

- Divinus Firmware direction: `work/fh8626v100-divinus@0b12c87c202b12733b0a1b535b56d66891e4ca93`; its only delta from core is the Divinus package selection.
- Majestic Firmware direction: `work/fh8626v100-majestic@7ed2a17a67fce0c2ee8d80bc798cafe81cfa6c38`; it adds the isolated FH8852V200 Majestic compatibility/control-plane package and a media-off configuration. This is staging, not an upstream-ready FH8626 media implementation.

The clean branch is `SOURCE_CONFIRMED / CLEAN_ARCH_CANDIDATE`, not a build or hardware acceptance. A production kernel-config audit removed only traced legacy/debug/dead facilities, made RTC device registration opt-in, and made recovery NFS explicitly v3-only. The exact decisions are in `docs/process/fh8626-kernel-config-audit.md`. The owner still needs to run the authoritative build and record the resolved `.config` plus final kernel/rootfs sizes. Binary ownership and exact hashes are recorded in `docs/process/fh8626-firmware-ownership-audit.md`.

### Builder

Repository: `ArthurKoba/openipc-builder`.

Historical Builder experiment:

- WIP commit: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
- parent/checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`
- no live Builder branch or tag is retained for this experiment;
- role: provenance only; the useful Majestic work has been reconstructed on the current clean Firmware/Builder staging branches.

Clean device-profile staging:

- branch: `work/fh8626v100-anjia`
- tip: `dac8d565aaa493c4fd83334df3054138b92ed01a`
- base: current Builder `master@e0a643f4942b064a149f470b3c118ebba4daebb5`
- kernel fragment: `CONFIG_FH8626V100_SD0_1BIT=y`

The staging branch contains only named-device deltas: ANJIA kernel fragment, RTL8188FU selection, microSD/device configuration, illumination helpers and source-built PTZ/lens support. It carries no generic FH8626 kernel config, no kernel patches, no factory `.ko/.so/.bin` and no Divinus source/patch.

Majestic Builder staging is `work/fh8626v100-anjia-majestic@85c496eab79e662cc2e0e511540265b969ae227a`. It adds a separate `fh8626v100_lite_anjia-ajl33pq0866_majestic` target and keeps it CI-opted-out while the required Firmware branch is fork-local. Builder now accepts `OPENIPC_FW_REPO` plus `OPENIPC_FW_REV` so this branch can be assembled without copying the Majestic package back into Builder.

This branch is `SOURCE_CONFIRMED / BUILDER_STAGING`, not a build or hardware acceptance. It is temporarily listed in Builder CI `NOT_BUILT` because the normal Builder flow still consumes `OpenIPC/firmware`; remove that opt-out only after the clean FH8626 Firmware integration is available from the Firmware source Builder consumes.

## Proprietary media/runtime retirement

The preserved Firmware branch still carries proprietary Fullhan runtime artifacts required by the historical working stack. They are transitional dependencies, not the target upstream architecture.

Current preservation inventory: eight kernel modules (`bgm.ko`, `enc.ko`, `gpio_wave.ko`, `isp.ko`, `jpeg.ko`, `media_process.ko`, `vmm.ko`, `xbus_rpc.ko`), two userspace plug-ins (`libmipi.so`, `libgc1054_mipi.so`) and six firmware/profile `.bin` objects including `rtthread_arc.bin` and GC1054 sensor/profile data.

The project already has substantial source-level ISP/AE/AWB/CCM/media-owner reconstruction. Reuse it rather than restarting ISP reverse wholesale, but do not equate userspace source coverage with replacement of the proprietary kernel modules. Retirement order and acceptance rules are in `docs/process/fh8626-blob-retirement.md`.

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

Majestic has a preserved hardware-proven **control-plane** result and a newly reconstructed staging path. The historical FH8852V200 binary successfully loaded on FH8626, served HTTP/WebUI on port 80, and remained stable when media consumers were disabled. Explicit GC1054 media initialization reached the SDK path and then crashed; direct I2C reads still returned GC1054 chip ID 0x10/0x54, so that experiment did not prove a dead sensor path.

Current staging is split correctly: Firmware `work/fh8626v100-majestic@7ed2a17...` owns the generic compatibility package; Builder `work/fh8626v100-anjia-majestic@85c496e...` owns the named-device selection. No FH8852 kernel modules or firmware are installed. See `docs/process/fh8626-majestic-staging.md`.

This is not completed Majestic media support. VI/VENC/RTSP/ISP/audio remain later compatibility work, and the moving donor `master` binary must be pinned/reproducible before any product acceptance.

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

1. reconcile the FH8626 kernel branch/upstream PR and build the final intended OpenIPC `uImage`;
2. measure that image against the standard 2 MiB kernel partition and audit configuration/compression only if it does not fit;
3. when matching kernel/rootfs images are ready, hardware-test the complete OpenIPC-native U-Boot/layout migration;
4. after hardware acceptance, curate the U-Boot history into a clean contribution series and update the stale workflow through an authorized identity;
5. classify the mixed Firmware preservation snapshot by repository ownership;
6. build and target-test the latest Divinus candidate until its FH8626 path is complete;
7. transition the product path to Majestic;
8. integrate shared runtime pieces into Firmware;
9. rebuild the final thin Builder device profile last.
