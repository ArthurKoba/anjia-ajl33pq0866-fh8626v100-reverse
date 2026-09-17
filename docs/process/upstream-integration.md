# Related repository integration

This document records the currently observed FH8626V100 working refs and the ownership rules to use when reconciling them. The refs are preservation/checkpoint locators, not automatic future bases or ready-to-submit contribution series.

Checked: `2026-09-17`.

Before using any ownership rule or preparing work for OpenIPC, read `openipc-upstream-rules.md` and re-open the relevant live upstream sources linked there. This file records project-specific refs and sequencing; `openipc-upstream-rules.md` records the external OpenIPC rules that can change independently of this project. Upstream repository-local instructions override cached summaries.

## Current observed refs

### U-Boot

- repository: `ArthurKoba/u-boot-fullhan`
- branch: `fh8626v100-mainline`
- observed tip: `ae63365e10b38e5b9ed3bd5173a0ed3e5c8f9996`
- repository `main`: `cc8c034e78eba6b2a3845783889b80753bb1af1e`
- relation: five FH8626V100 commits ahead of `main`

The branch is a modern upstream-U-Boot-based FH8626V100 port and is already hardware-used on ANJIA AJL33PQ0866. The 2026-09-17 audit found no missing capability required by the exercised OpenIPC boot/recovery/update path. Its remaining job is contribution curation and handoff, not reimplementation.

Important handoff boundaries:

- the current artifact is board-specific and must not be advertised as universal FH8626V100;
- the Fullhan ROM envelope is `0x2bb00`; the audited production build has only 820 bytes of spare room;
- U-Boot, Linux and the preserved FH8626 Firmware build agree on kernel `0x50000`, `rootfs_data` `0x350000`, rootfs `0x450000`;
- current generic OpenIPC image assembly uses different ordinary rootfs offsets, so FH8626 needs an explicit assembly path;
- OpenIPC currently uses multiple SoC/family-specific U-Boot repositories. No current OpenIPC Fullhan U-Boot source repository was identified, so maintainers should choose the source-repository ownership before organization-level publication.

Recommended source handoff: curate a clean series in `u-boot-fullhan`, provide hardware/migration/provenance evidence, ask OpenIPC maintainers whether they want a new Fullhan/FH8626 repository or another destination, then integrate the agreed board-specific release artifact into Firmware. Do not move U-Boot source into Firmware or Builder.

Detailed technical audit: `docs/hardware/uboot-port.md`.

### Linux/kernel

- repository: `ArthurKoba/openipc-linux`
- branch: `fullhan-fh8626v100`
- observed tip: `ebf5d776c748edbd58c1aaf8be9d5b2639a16834`
- parent lineage used by the FH8626 series: `fullhan-fh8852v200`
- FH8626 series: two commits

The operator reports that FH8626V100 kernel support has already been submitted to upstream OpenIPC. Before changing or curating this branch, locate the exact upstream pull request and verify its current head, status, review comments and relation to the local branch.

As checked on 2026-09-17, the public `OpenIPC/linux` branch table does not yet list FH8626V100, so local branch existence must not be confused with completed upstream integration.

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

The WIP is intentionally treated as a recovery snapshot. It currently includes multiple ownership domains in one commit: kernel patches/config, board-specific support, Divinus patching, proprietary Fullhan modules/libraries, and a large camera/media source/test tree. Do not use its current directory placement as architectural authority.

### Builder

- repository: `ArthurKoba/openipc-builder`
- branch: `fh8626v100-anjia-ajl33pq0866`
- observed tip: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
- stable pre-Majestic checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`
- preceding Divinus/device integration commit: `c7577cb3f70b4531e9ec9e686c5275bf5f170c3d`
- current upstream `master` has advanced beyond the preserved branch; the branch is diverged

`5603a701...` is the deliberate rollback/reference point before the Majestic experiment. The next commit, `bcf8658e...`, adds the preserved Majestic experiment. Do not combine these two states when reasoning about the Divinus baseline.

Future Builder work must start by reconciling against the then-current upstream `master`. These preserved SHAs are evidence/reference points, not future contribution bases.

## Ownership routing

The current OpenIPC repository rules and this project's architecture imply the following destination table. Revalidate the OpenIPC side against `openipc-upstream-rules.md` before acting on it.

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
| FH8626 U-Boot source | `ArthurKoba/u-boot-fullhan` until upstream ownership is explicitly chosen |
| Mutable reverse-analysis state | canonical Ghidra MCP project |
| Heavy/unique primary evidence | project Google Drive evidence store |

Firmware's current rules explicitly route kernel patches to `OpenIPC/linux` and support for one retail camera model to `OpenIPC/builder`. Builder's current rules describe it as a thin per-device overlay and state that reusable/common code belongs in Firmware or the component repository that owns it. These are externally maintained rules and must be refreshed before contribution work.

## Reconciliation method

For each repository:

1. read `openipc-upstream-rules.md` and refresh the relevant live upstream sources;
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

The current execution order is:

1. U-Boot contribution curation/handoff from the completed audit;
2. Linux/kernel audit and upstream reconciliation;
3. Firmware ownership/placement audit;
4. latest Divinus build + physical-camera acceptance and completion;
5. Majestic product transition;
6. shared Firmware product integration;
7. Builder final device profile.

This order intentionally keeps Builder last and prevents the old preservation snapshots from dictating the final architecture.
