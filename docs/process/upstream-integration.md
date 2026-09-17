# Related repository integration

This document records the currently observed FH8626V100 working refs and the ownership rules to use when reconciling them. The refs are preservation/checkpoint locators, not automatic future bases or ready-to-submit contribution series.

Checked: `2026-09-17`.

Before using any ownership rule or preparing work for OpenIPC, read `openipc-upstream-rules.md` and re-open the relevant live upstream sources linked there. This file records project-specific refs and sequencing; `openipc-upstream-rules.md` records external OpenIPC rules that can change independently. Repository-local upstream instructions override cached summaries.

## Current observed refs

### U-Boot

Repository: `ArthurKoba/u-boot-fullhan`.

Hardware-proven recovery/reference branch:

- `fh8626v100-mainline`
- observed tip: `49fe46e9ddb786e232d1359f9cee68c914a3a8db`
- repository `main`: `cc8c034e78eba6b2a3845783889b80753bb1af1e`

Implemented OpenIPC-native candidate:

- `fh8626v100-openipc-native`
- observed tip: `99c477674acc250b3177e3ae6eaaa1680c28c809`
- five commits ahead of the hardware-proven baseline
- current evidence level: source/build candidate, not hardware acceptance

The native branch targets standard OpenIPC 8 MiB NOR geometry rather than the historical Fullhan partition map:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.

The Fullhan-specific board requirement is contained inside the normal 256 KiB `boot` partition: 64 KiB reconstructed Boot-ROM/DDR data followed by a 192 KiB U-Boot slot. The persistent environment is at `0x40000`, kernel at `0x50000`, rootfs at `0x250000` and rootfs_data at `0x750000`.

The native production target uses OpenIPC environment/update conventions and no longer depends on the factory Fullhan environment. Factory `kload` and vendor GPIO syntax remain opt-in only in the RAM migration target.

Observed native build result at `99c47767...`:

- raw production U-Boot: `0x2ed18`;
- physical U-Boot slot: `0x30000`;
- free payload budget after four-byte checksum fixup: 4836 bytes;
- production and RAM targets both compile;
- native board boot artifact generation succeeds.

The current repository workflow reports red only because its final post-build commands still check the old stock artifact filenames. The available GitHub App is denied workflow-file mutation permission, so that workflow update must be performed later by an identity with GitHub workflow-write access. Do not work around that permission boundary or preserve obsolete artifacts only to satisfy the stale check.

The immediate U-Boot gate is hardware, not additional architecture work: first prove the final FH8626 kernel fits the standard OpenIPC 2 MiB partition, then perform the documented native full-layout migration and cold boot. Keep `fh8626v100-mainline` untouched as recovery authority until that passes.

OpenIPC currently uses multiple SoC/family-specific U-Boot repositories. No OpenIPC Fullhan U-Boot repository was identified during the current check. After hardware acceptance, curate/squash the native series, re-run current U-Boot contribution gates, provide provenance/evidence, and ask OpenIPC maintainers which source repository should own the port. Do not move U-Boot source into Firmware or Builder.

Detailed technical state: `docs/hardware/uboot-port.md`.

### Linux/kernel

- repository: `ArthurKoba/openipc-linux`
- branch: `fullhan-fh8626v100`
- observed tip: `ebf5d776c748edbd58c1aaf8be9d5b2639a16834`
- parent lineage used by the FH8626 series: `fullhan-fh8852v200`
- FH8626 series: two commits

The operator reports that FH8626V100 kernel support has already been submitted to upstream OpenIPC. Before changing or curating this branch, locate the exact upstream pull request and verify its current head, status, review comments and relation to the local branch.

As checked on 2026-09-17, the public `OpenIPC/linux` branch table does not yet list FH8626V100, so local branch existence must not be confused with completed upstream integration.

The first kernel reconciliation output must include the exact final `uImage` size because the OpenIPC-native U-Boot target now assumes the standard 2 MiB kernel partition.

### Divinus

- repository: `ArthurKoba/openipc-divinus`
- branch: `fh8626v100-canonical`
- observed tip: `1e624bd5aca97ba772413d2b00a10314d1db039f`
- generic FH8626 integration commit: `8d400262898e8e82df6171fde7e8911ec7930249`
- preservation commit: `1e624bd5aca97ba772413d2b00a10314d1db039f` (`WIP: preserve FH8626V100 native HAL migration state`)

The preservation commit contains the newest native-HAL/media/ISP/audio/transport work. It is not yet hardware acceptance. Build and target-test this exact latest source before deciding what to keep or change.

### Firmware

- repository: `ArthurKoba/openipc-firmware`
- branch: `fh8626v100-platform`
- observed tip: `6db66c53971fda8ba733f370a965e52cd53fb61b`
- upstream/base at preservation time: repository `master`
- preservation shape: one WIP commit

The WIP is a recovery snapshot. It currently includes multiple ownership domains in one commit: kernel patches/config, board-specific support, Divinus patching, proprietary Fullhan modules/libraries, and a large camera/media source/test tree. Do not use its current placement as architectural authority.

The historical FH8626-specific 3 MiB kernel / rootfs-at-`0x450000` layout is also preservation state. If the final kernel meets the 2 MiB target, Firmware should converge to ordinary OpenIPC 8 MiB image boundaries.

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

1. build/measure final FH8626 kernel and hardware-accept the implemented OpenIPC-native U-Boot/layout;
2. curate/handoff U-Boot after that hardware gate;
3. complete kernel upstream reconciliation;
4. perform Firmware ownership/placement sanitation;
5. build + hardware-accept latest Divinus;
6. transition to Majestic product path;
7. integrate shared Firmware product pieces;
8. build the final thin Builder device profile last.

This order keeps the lowest platform contracts OpenIPC-native before higher layers are finalized and prevents preservation snapshots from dictating product architecture.
