# Current project state

Status: `ACTIVE / UBOOT_NATIVE_BUILD_READY / KERNEL_STAGING / PINNED_MEDIA_RUNTIME / MAJESTIC_OFFLINE_CLOSURE / OWNER_BUILD_GATE`.

Checked: `2026-09-18`.

## Authority

- GitHub owns current camera-level source, documentation, contracts, state and evidence manifests.
- Google Drive owns heavy or unique primary evidence referenced by SHA-256 from `evidence/MANIFEST.tsv`.
- Ghidra MCP through Koba MCP Bridge is the canonical mutable reverse-analysis workspace.
- GC1054/MIPI userspace reverse is consolidated into the existing Apollo Ghidra project `anjia_ajl33pq0866_fh8626v100_apollo` at `/projects/anjia-ajl33pq0866-fh8626v100/anjia_ajl33pq0866_fh8626v100_apollo.gpr`; the programs are `/sensor/libgc1054_mipi.so` and `/sensor/libmipi.so`. The former standalone `sensor_libs.gpr` duplicate was deleted after migration. Headless project discovery must search `/projects` explicitly.

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
- current candidate: `44c4fb94c5a021695c18123c6c703fdf0c79cb3c`
- evidence class: `SOURCE_CANDIDATE / TARGET_PENDING`

The Divinus agent and Majestic agent now share recovered contracts through reverse
issue #3. Current Divinus already incorporates the critical Apollo/Ghidra
corrections found during the Majestic clean-room pass: VPU enable receives the
channel id rather than a boolean, VPU disable is the distinct request, VI/VPU
ownership is separated from VENC start, and the VENC startup order is
StartRecvPic -> force-I -> media bind. H.264 RC defaults, fixed-QP/CVBR and
JPEG/MJPEG wire handling were also tightened against the same recovered
contracts. Divinus now also carries the native GraphV2 OSD backend. Its
selector/slot limits were independently rechecked against Ghidra `isp.ko` and
then applied to the Majestic facade as a shared FH8626 contract.

The implementation is broader than the current Builder acceptance YAML. The
Builder Divinus profile intentionally keeps audio and JPEG/MJPEG disabled for
the first target gate; that is test sequencing, not missing repository
ownership.

No Divinus result is a substitute for the canonical Apollo/kernel evidence.
New disagreements between the two runtime implementations must be recorded in
issue #3 before either implementation becomes the new reference.

### Firmware

Repository: `ArthurKoba/openipc-firmware`.

Preservation/evidence snapshot:

- tag: `archive/fh8626v100-platform-20260918`
- commit: `f4bf49da6ef355c9e733e00d774efe403513b1d4`
- role: immutable historical source of the still-proprietary media kernel/ARC
  payloads plus old mixed implementation evidence

Active directions:

- core: `work/fh8626v100@80169887be80c43471f9f4792dde3e2a5bd18a8c`
- Divinus: `work/fh8626v100-divinus@255b8c8deea8e7da5ef7429b6f8f2b176a430aec`
- Majestic: `work/fh8626v100-majestic@04e093605fbd18706f69c4d3363cb508e08bfcf1`

All three directions consume the exact Linux staging source
`openipc-linux/work/fh8626v100@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`.
The standard OpenIPC 8 MiB budget is 2048 KiB kernel and 5120 KiB SquashFS,
matching the native U-Boot map.

### Proprietary media-kernel runtime ownership

A pre-deploy audit found that the earlier source cleanup removed the checked-in
Fullhan media blobs correctly but also removed the build-time owner for their
still-required kernel ABI. That gap is now closed by generic package
`fullhan-media-fh8626v100`, selected by the FH8626 base defconfig in all
three runtime directions.

Active source branches do **not** contain the binary payloads. Buildroot
downloads exactly nine pinned files from the immutable preservation commit and
verifies every one by SHA-256:

- `bgm.ko`
- `enc.ko`
- `gpio_wave.ko`
- `isp.ko`
- `jpeg.ko`
- `media_process.ko`
- `vmm.ko`
- `xbus_rpc.ko`
- `rtthread_arc.bin`

Independent immutable copies of those same bytes were also ingested into Koba
artifact storage. The package installs no archived sensor plug-in or archived
`libmipi.so`; those userspace boundaries are owned by current runtime
implementations.

The installed `load_fullhan -i` integrates with standard OpenIPC
`S70vendor` and preserves the historical load order:
VMM -> XBUS/ARC -> media_process -> ISP -> encoder -> JPEG -> BGM ->
gpio_wave. It fails unless the required media character devices are created.

The eight modules report `vermagic=4.9.129 mod_unload ARMv6 p2v8`, matching
the target kernel family/config ABI. The current Linux staging commits alter
platform/GMAC/PWM/RTC/USB areas, not the proprietary media module ABI. This is
a strong static compatibility check, not a replacement for target module-load
acceptance.

Raw proprietary media/ARC payload size is about 798 KiB before SquashFS. The
5120 KiB rootfs budget therefore remains an explicit owner-build gate.

This package is transitional deployment infrastructure, not blob retirement.
The final architecture still aims to replace the opaque kernel/ARC pieces with
reproducible source implementations.

### Builder

Repository: `ArthurKoba/openipc-builder`.

- active branch: `work/fh8626v100-anjia@ee0687c06b2d80285defe7da14596041ffd221ca`
- one physical device tree with composed `_divinus`, `_majestic` and
  `_diag` variants
- no live separate Majestic Builder branch

Composition is:

`Firmware fh8626v100_lite_defconfig -> ANJIA base.config -> runtime fragment`.

Sibling `.firmware` metadata chooses the correct Firmware direction
automatically:

- Divinus -> `work/fh8626v100-divinus`
- Majestic -> `work/fh8626v100-majestic`
- diag -> `work/fh8626v100`

Builder remains thin. It does not store proprietary FH8626 media binaries.
Board-local packages own PTZ, lens, illumination/IR-cut/SADC policy,
speaker-amplifier mute, storage, Wi-Fi selection and persistent target/update
identity.

The device README was refreshed during the pre-deploy audit and is now the
operator-facing build/deployment checklist. The current variants remain
`NOT_BUILT`; no current composed image size or hardware pass may be inferred
from historical builds.

## Proprietary media/runtime retirement

The active Firmware/Builder branches no longer carry opaque FH8626 media bytes.
The preservation tag
`openipc-firmware/archive/fh8626v100-platform-20260918@f4bf49da...`
remains the immutable provenance source.

For pre-deploy testing, the still-required kernel/ARC runtime is now selected
through `fullhan-media-fh8626v100`: package metadata, loader scripts and
SHA-256 manifests live in active source, while the nine opaque payloads are
fetched from the immutable preservation commit and independently mirrored in
Koba artifact storage.

This fixes build ownership without pretending blob retirement is complete.
The proprietary runtime is still technical debt. Kernel source currently does
not contain open replacements for `isp.ko`, `enc.ko`, `jpeg.ko`,
`media_process.ko`, `vmm.ko`, `xbus_rpc.ko`, `bgm.ko` or
`gpio_wave.ko`; `rtthread_arc.bin` also remains opaque.

Majestic has already retired the active vendor GC1054 and MIPI userspace path
with source implementations. Do not reintroduce the archived sensor/MIPI blobs
into the deployment package.

Retirement order and evidence rules remain in
`docs/process/fh8626-blob-retirement.md`.

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

Divinus remains the open reference path that must be closed before the Majestic product transition. The native FH8626 implementation has now been source-cleaned to `168b2ec...`: obsolete external-owner transport is retired, native ownership/IDR/telemetry/lifecycle boundaries are explicit, and unsafe runtime MP4 mutation is blocked instead of falling through generic HAL code.

The candidate still requires the focused host suite, exact ARM1176/musl Firmware build and physical-camera acceptance. Its current transitional sensor plug-in and RTX helper dependencies are explicit blockers, not hidden production assumptions.

## Majestic

Majestic remains the product-focus runtime. Historical hardware evidence proves
its control plane: the FH8852V200 Lite binary ran on FH8626 and served the
native HTTP/WebUI when media was disabled. The old explicit sensor path then
segfaulted because multiple FH8852/FH8626 ABI boundaries were still wrong.

Current Firmware direction:
`work/fh8626v100-majestic@04e09360...`.

The offline compatibility closure is now substantially reconstructed rather
than a fixed 720p bring-up shim:

- source GC1054 and MIPI path;
- FH8852 sensor-table -> FH8626 callback translation;
- source VMM facade;
- source SYS/VPSS/VENC/stream translation;
- main/sub/analytics H.264 channel ownership;
- profile, GOP, visible geometry, full stock RC modes, readback and realtime RC;
- exact encoded FIFO/channel/lease translation;
- native JPEG/MJPEG public surface;
- motion YCmean/CPY backend;
- OSD GraphV2 backend, including Ghidra-confirmed native selector/slot limits
  (global slots 0..1, channel slots 0..3);
- source RTX/ACW audio including retail DSP init, AEC/NR/AGC extensions and
  board-neutral AO lifecycle hook;
- recovered ANJIA day/night GPIO contract in the strict full profile;
- direct + transitive ABI guard that rejects a moving Majestic/vendor update if
  it begins depending on an unsupported SDK boundary.

HEVC/H.265 is explicitly unsupported because the retained FH8626 encoder stack
does not register an HEVC engine. Unsupported SDK-only exports are errors, not
fake success.

The retained donor ISP/ispcore/advapi layer stays isolated as one coherent
userspace ISP context; replacing isolated pieces with guessed direct ioctls
would reduce correctness.

Majestic HTTP/WebUI remains untouched. No auxiliary proxy and no JavaScript
patch are accepted. The historical empty `/metrics` result remains a separate
provider/backend question.

Default boot stays media-off. `majestic-fh8626-full-run` is the strict target
acceptance profile: native VENC enabled, permissive stubs disabled, main+sub
H.264, JPEG, OSD, motion, audio, RTSP and board day/night enabled together.

The offline selected runtime closure is complete enough to stop speculative
adapter work. The Majestic compatibility package now explicitly selects the
shared pinned FH8626 media runtime, so that dependency cannot disappear through
a variant/config composition mistake. The next blocker is now an **actual
composed build**: verify the media package downloads/hashes, require
`uImage <= 2048 KiB` and `rootfs.squashfs <= 5120 KiB`, record the exact
downloaded Majestic executable SHA-256, then test on the camera. The FH8852
Majestic executable remains a moving upstream `lite.master` object and is a
product-reproducibility boundary even if the controlled first build succeeds.

A successful build is still not product acceptance. Same-boot/reconfigure,
media devices, RTSP/JPEG/OSD/motion/audio/day-night/PTZ/storage and persistent
boot/update behavior remain hardware gates.

## Audio

RTX microphone and speaker paths are hardware-proven on this board. Product-streamer integration remains to be validated in the selected final path.

## PTZ and illumination

The accepted low-level PTZ backend is `/dev/fh_pwm`. Builder now uses it as a stateless relative OpenIPC backend rather than a boot-calibrated coordinate controller; that architecture is source-checked but still needs target regression. The stock-style calibration/persistent-state implementation is preserved by immutable Builder tag for reference only.

Illumination/IR-cut board contracts remain streamer-independent. Builder now exposes only physical board helpers plus safe LED/IR-cut-rest handling. AUTO/day/night/WLIGHT scene policy stays with the media runtime. The staged helper currently preserves its pre-cleanup IR-cut active/rest drive values pending a target direction/polarity regression; this must not be confused with the stock configuration's logical polarity field.

## Curated source

Reusable camera-specific engineering components are retained under `source/fh8626v100/components/`. They are implementation references/contracts, not permission to duplicate the same source into Firmware, Builder and Divinus.

## Evidence

The retained GC1054 vendor sensor binary is external evidence:

- SHA-256: `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`
- role/locator: `evidence/MANIFEST.tsv`

Firmware, dumps and other heavy primary evidence remain external and SHA-addressed.

## Immediate engineering sequence

1. Owner-build
   `fh8626v100_lite_anjia-ajl33pq0866_majestic` from Builder
   `work/fh8626v100-anjia`; retain the archived composed config and resolved
   Builder/Firmware/Linux SHAs.
2. Verify all nine `fullhan-media-fh8626v100` downloads pass SHA-256 and the
   final target contains the eight modules, `rtthread_arc.bin`,
   `fh-media-modules` and `load_fullhan`.
3. Record exact image sizes. Hard gates: `uImage <= 2048 KiB`,
   `rootfs.squashfs <= 5120 KiB`; also record remaining headroom.
4. For the first Majestic media test, avoid combining it with the new native
   U-Boot migration unless the existing bootloader cannot boot the matching
   kernel/rootfs layout. The new U-Boot mainline remains a separate cold-boot
   hardware gate.
5. Boot the default media-off image and verify board/network/control-plane
   baseline. Confirm `S70vendor` creates `/dev/vmm_userdev`,
   `/dev/media_process`, `/dev/isp`, `/dev/pae` and `/dev/jpeg`.
6. Run `majestic-fh8626-abi-probe`; retain complete output and exact Majestic
   build identity.
7. Run `majestic-fh8626-full-run` and validate simultaneous main/sub RTSP,
   live RC/GOP/readback, JPEG, OSD, motion, capture/playback audio and
   day/night/IR-cut.
8. Regress repeated stop/start/reconfigure, lens, PTZ, storage, Wi-Fi, reset,
   shutdown and board-safe GPIO states.
9. Only after media/runtime acceptance decide whether to migrate the same test
   image onto `u-boot-fullhan/fh8626v100-mainline`; perform that migration
   with full-flash backup and external SPI recovery available.
10. Feed every target finding back into reverse issue #3 when it changes a
    shared FH8626 contract used by Majestic and Divinus.
