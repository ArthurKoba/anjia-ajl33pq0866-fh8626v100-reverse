# Related repository integration

This document records the currently observed FH8626V100 working refs and the ownership rules to use when reconciling them. The refs are preservation/checkpoint locators, not automatic future bases or ready-to-submit contribution series.

Checked: `2026-09-18`.

Before using any ownership rule or preparing work for OpenIPC, read `openipc-upstream-rules.md` and re-open the relevant live upstream sources linked there. This file records project-specific refs and sequencing; `openipc-upstream-rules.md` records external OpenIPC rules that can change independently. Repository-local upstream instructions override cached summaries.

## Current observed refs

### U-Boot

Repository: `ArthurKoba/u-boot-fullhan`.

Current OpenIPC-native working line:

- `fh8626v100-mainline`
- observed tip: `227bcb40f68147864d778f1431973566cca383d8`
- equivalent native implementation ref: `fh8626v100-openipc-native@227bcb40...`
- repository upstream base `main`: `cc8c034e78eba6b2a3845783889b80753bb1af1e`
- current evidence level: source/build accepted, native hardware acceptance pending

Preserved hardware-proven factory-compatible state:

- `fh8626v100-stock-compatible@49fe46e9ddb786e232d1359f9cee68c914a3a8db`

The old proven state was given a dedicated recovery/evidence branch before `fh8626v100-mainline` was fast-forwarded to the agreed OpenIPC-native implementation. Do not confuse the new current working line with physical acceptance of the relocated flash layout.

The current U-Boot target uses standard OpenIPC 8 MiB NOR geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.

Board-specific Fullhan requirements are contained inside the normal 256 KiB boot partition: 64 KiB reconstructed Boot-ROM/DDR data at `0x00000` and a 192 KiB physical U-Boot slot at `0x10000`. Environment is at `0x40000`, kernel at `0x50000`, rootfs at `0x250000`, and rootfs_data begins at `0x750000`.

Production uses normal OpenIPC NOR environment naming: `kernaddr/kernsize`, `rootaddr/rootsize`, `mtdpartsnor8m`, `setnor8m`, `cmdnor`, `bootcmdnor`, `updatetool`, `ubnor/ubwrite`, `uknor/ukwrite`, `urnor/urwrite`. Factory `kload` and vendor GPIO syntax are opt-in RAM-recovery compatibility only.

U-Boot update uses the board-qualified artifact `u-boot-fh8626v100-anjia-ajl33pq0866-nor.bin`, not a universal FH8626 filename. The artifact is exactly `0x50000` bytes: the 256 KiB boot partition plus an erased 64 KiB environment sector, matching the normal OpenIPC updater boundary at the kernel offset.

The production Fullhan descriptor is payload-derived rather than stock-fixed. Observed build at `227bcb40...`:

- raw U-Boot `0x2ef20`;
- aligned ROM payload `0x2f000`;
- JAMCRC `0x0c6d419b`;
- flash source `0x10000`;
- load/entry `0xa0800000`;
- physical slot `0x30000`;
- boot artifact `0x40000`;
- NOR updater artifact `0x50000`.

`build.sh` ran 8 native artifact tests, compiled production, generated/self-inspected the native NOR artifact, then compiled the RAM recovery target successfully. The GitHub workflow still reports red only after those successful stages because its final stock-era post-check references removed legacy artifact names/descriptor values. The permitted Bridge App lacks `workflows` write permission, so the check definition cannot currently be fixed through the authorized path. Do not fake old artifacts or bypass the permission boundary.

The immediate U-Boot gate is now physical integration, not further stock-layout design. First reconcile/build the final kernel and prove the `uImage` fits the standard 2 MiB partition; then perform the documented native full-layout migration and cold boot with external recovery available.

After native hardware acceptance, rebuild/squash the iterative U-Boot implementation history into a clean contribution series and run current U-Boot style/checkpatch gates. OpenIPC currently uses multiple SoC/family U-Boot repositories and no OpenIPC Fullhan U-Boot repository has been established for this work. Present the proven source/evidence/provenance/artifact scheme to maintainers and let them choose organization-level ownership. Do not move U-Boot source into Firmware or Builder.

Detailed technical state: `docs/hardware/uboot-port.md`.

### Linux/kernel

- repository: `ArthurKoba/openipc-linux`
- verified base: `fullhan-fh8852v200@ee1ef844294bfa1ff15b2f0522d35c987a16a220`
- PR-facing/integration branch: `fullhan-fh8626v100@0dfafa643770d78389e444c03f46f1711662eda6`
- isolated MTD topic: `fix/fh8626v100-openipc-mtd-layout@28a923a9d9598a9a4e2c6c0ee4b2eee26698731e`
- exploratory reconstruction: `rework/fh8626v100-clean-series@868bdd8ddde7a35c2c744e5706941d5e1f9faadf`
- curated staging series: `rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`

The previous `ebf5d776...` locator is stale after identity rewrite/repoint. The live integration branch is `0dfafa64...`.

The two-commit Linux migration has been reconstructed into a 13-commit subsystem-oriented staging series. Findings include the MTD mismatch, mandatory Fullhan boardconfig copy input, board-profile pin selection, neutral one-bit SD0 Kconfig naming, missing PWM v2 Makefile wiring, non-DT AXI-DMA platform registration, independently reviewable clock/pinctrl/PWM/RTC/DWC2 fixes, JL1101 support, MAC propagation and checksum-feature correction. Hardware-tested static RMII behavior is deliberately preserved.

The historical accepted `uImage` was 1,583,456 bytes and proves the tested tree fit 2 MiB. The curated `final-series` is source-reviewed but still awaits the owner's authoritative build/check/hardware gates.

As rechecked on 2026-09-18, the public OpenIPC/linux branch list still does not show FH8626V100. The operator reports an upstream submission exists, but the exact PR number/status is not independently verified through the currently accessible API.

Detailed audit: `docs/process/fh8626-kernel-series-audit.md`.

### Divinus

- repository: `ArthurKoba/openipc-divinus`
- work branch: `work/fh8626v100`
- source-clean candidate: `168b2ecfeffcb53c2ed2a1d86c4897fdd3423820`
- preserved migration input: `1e624bd5aca97ba772413d2b00a10314d1db039f`
- generic integration commit: `8d400262898e8e82df6171fde7e8911ec7930249`

The current candidate removes the obsolete external FH86 owner/source protocol and keeps generic FH8626 ownership in Divinus native HAL code. Board hooks/policy are excluded. Native force-IDR, best-effort lifecycle cleanup and platform/media telemetry are wired. FH8626 temperature is explicitly unsupported.

The candidate is source-cleaned but not hardware-accepted. Current blockers are the vendor GC1054/MIPI plug-in dependency, external RTX audio-helper dependency, full runtime video-reconfiguration, same-boot teardown/restart acceptance and exact-candidate target acceptance. Focused host tests are grouped by `tests/fh8626-check.sh` and remain an owner/CI execution gate.

The matching Firmware hardware-test direction is `work/fh8626v100-divinus@d589e0546384a4fc094cc959f81d6f2864ce997f`. It temporarily pins exactly `ArthurKoba/openipc-divinus@168b2ec...` rather than moving upstream `HEAD`. This is staging provenance only; upstream-ready Firmware must consume OpenIPC-owned Divinus source.

### Firmware

- repository: `ArthurKoba/openipc-firmware`
- branch: `fh8626v100-platform`
- observed tip: `f4bf49da6ef355c9e733e00d774efe403513b1d4`
- upstream/base at preservation time: repository `master`
- preservation shape: one WIP commit

The WIP is a recovery snapshot. It currently includes multiple ownership domains in one commit: kernel patches/config, board-specific support, Divinus patching, proprietary Fullhan modules/libraries, and a large camera/media source/test tree. Do not use its current placement as architectural authority.

The proprietary runtime is tracked as retirement debt rather than an accepted permanent Firmware package. The preserved snapshot contains eight media `.ko` files, `libmipi.so`, `libgc1054_mipi.so`, `rtthread_arc.bin` and GC1054 binary/profile data. See `docs/process/fh8626-blob-retirement.md`.

The historical FH8626-specific 3 MiB kernel / rootfs-at-`0x450000` layout is preservation state only. If the final kernel meets the 2 MiB target, Firmware must converge to the ordinary OpenIPC 8 MiB image boundaries already implemented in U-Boot.

### Builder

- repository: `ArthurKoba/openipc-builder`
- branch: `fh8626v100-anjia-ajl33pq0866`
- observed tip: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
- stable pre-Majestic checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`
- preceding Divinus/device integration commit: `c7577cb3f70b4531e9ec9e686c5275bf5f170c3d`
- current upstream `master` has advanced beyond the preserved branch; the branch is diverged

`5603a701...` is the deliberate rollback/reference point before the Majestic experiment. The next commit, `bcf8658e...`, adds the preserved Majestic experiment. Do not combine these states when reasoning about the Divinus baseline.

Future Builder work must start from then-current upstream `master`. Preserved SHAs are evidence/reference points, not future contribution bases.

## Ownership routing

Revalidate this table against `openipc-upstream-rules.md` before acting on it.

| Change/artifact | Owning repository |
|---|---|
| Camera-level hardware/media facts, evidence boundaries, target contracts | `anjia-ajl33pq0866-fh8626v100-reverse` |
| Kernel source, platform support, kernel patches | `OpenIPC/linux` contribution path |
| Kernel configuration selected by the OpenIPC image build | `OpenIPC/firmware` where appropriate |
| Divinus HAL/media/transport implementation | Divinus owning repository |
| Majestic implementation bugs/features | Majestic owning repository/maintainers |
| Shared SoC-family packages, load scripts, runtime integration, rootfs infrastructure | `OpenIPC/firmware` |
| One specific retail camera/device profile and per-device deltas | `OpenIPC/builder` |
| Probing/bring-up tooling intended for OpenIPC | `OpenIPC/ipctool` where appropriate |
| FH8626 U-Boot source | `ArthurKoba/u-boot-fullhan` until OpenIPC ownership is explicitly chosen |
| Mutable reverse-analysis state | canonical Ghidra MCP project |
| Heavy/unique primary evidence | project Google Drive evidence store |

Firmware's current rules explicitly route kernel patches to `OpenIPC/linux` and support for one retail camera model to `OpenIPC/builder`. Builder is a thin per-device overlay. These are externally maintained rules and must be refreshed before contribution work.

## Reconciliation method

For each repository:

1. read `openipc-upstream-rules.md` and refresh relevant live upstream sources;
2. inspect the current repository and current upstream/base before editing;
3. identify the preserved FH8626 state and distinguish hardware-proven behavior from source-only/WIP work;
4. compare it against current camera contracts in this repository;
5. classify every changed file by owning repository before moving/deleting anything;
6. preserve unique evidence and working behavior even when current placement is wrong;
7. remove generated binaries, debug scaffolding and obsolete experiments only after their role/provenance is understood;
8. rebuild accepted changes on a clean working branch from the verified target base;
9. group work into coherent commits owned by that repository;
10. build/test at the appropriate evidence level;
11. re-read current contribution/PR gates before declaring the series upstream-ready;
12. leave final pull-request creation to the repository owner.

## Active order

1. reconcile the kernel/upstream PR and build/measure the final FH8626 OpenIPC kernel;
2. hardware-accept the already implemented OpenIPC-native U-Boot/layout with matching kernel/rootfs images;
3. curate/handoff U-Boot after that hardware gate;
4. perform Firmware ownership/placement sanitation;
5. build + hardware-accept latest Divinus;
6. transition to Majestic product path;
7. integrate shared Firmware product pieces;
8. build the final thin Builder device profile last.

This order keeps the lowest platform contracts OpenIPC-native before higher layers are finalized and prevents preservation snapshots from dictating product architecture.
