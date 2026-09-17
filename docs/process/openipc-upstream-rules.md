# OpenIPC upstream ownership and contribution rules

Status: `CURRENT EXTERNAL-RULE SNAPSHOT`.

Checked against live OpenIPC sources: `2026-09-17`.

This document is the local project summary of current OpenIPC repository boundaries, contribution rules and review gates relevant to the FH8626V100 / ANJIA AJL33PQ0866 work.

It is intentionally **not** a substitute for upstream documentation. OpenIPC changes over time. Before modifying an OpenIPC-related repository or preparing an upstream contribution, re-open the live sources listed below and verify that this summary is still correct.

## Mandatory refresh rule

Before any implementation or contribution work in `openipc-linux`, `openipc-firmware`, `openipc-builder`, Divinus, Majestic integration, or an OpenIPC U-Boot repository:

1. read this file;
2. open the relevant live upstream rules/source links in the `Live sources` section;
3. inspect the current repository `README`, `AGENTS.md` / `CLAUDE.md`, contribution/review files and current target branch;
4. verify that repository ownership, branch naming, build/test requirements and hard gates have not changed;
5. if upstream rules differ from this document, treat upstream as authoritative and update this file in the same work cycle before continuing;
6. do not rely on an old commit SHA, branch list, board count or package layout merely because it is recorded here.

When preparing a contribution, record which upstream branch/rules were rechecked and the date of the check in the working notes or contribution preparation record.

## Repository ownership

### Kernel / Linux

Kernel source, device-tree changes, kernel drivers and new kernel patches belong in:

`https://github.com/OpenIPC/linux`

OpenIPC Firmware explicitly redirects kernel source and kernel patches to `OpenIPC/linux`. New kernel patches are not supposed to be introduced as firmware-side patch files. A patch accepted into `OpenIPC/linux` should not also be duplicated in Firmware.

The Firmware-side exception is **kernel configuration** for a board/family. Kernel config files selected by Firmware/Buildroot remain part of the Firmware build description; changing a config symbol is different from changing kernel source.

The current `OpenIPC/linux` repository is organized primarily by manufacturer/processor branches rather than one universal product branch. As checked on 2026-09-17, its public branch table contains Fullhan branches for FH8833V100, FH8852V100 and FH8852V200 families, but does not yet list FH8626V100. Therefore our `ArthurKoba/openipc-linux/fullhan-fh8626v100` branch must not be treated as already upstream solely because it exists in our fork.

### Firmware

`https://github.com/OpenIPC/firmware`

Firmware is a Buildroot external tree. Current upstream rules say this repository owns shared build/integration material such as:

- shared packages;
- SoC-family drivers and load scripts;
- root filesystem overlay that is genuinely generic;
- SoC/family board defconfigs and build-system integration;
- package selection and shared runtime integration.

Firmware is **not** the correct home for:

- new kernel source/driver patches — use `OpenIPC/linux`;
- support that is specific to one retail camera — use `OpenIPC/builder`;
- probing/register-poking/bring-up utilities — use `OpenIPC/ipctool`;
- a workaround for a Majestic bug — fix/report it with Majestic maintainers rather than hiding it in Firmware.

The shared-tree blast radius is a first-class review concern. Values true only for one physical camera — GPIOs, sensor names, I2C addresses, MAC prefixes, fixed resolutions, device IPs — must not be placed into generic `general/overlay/` or made the default of a family-wide `load_<vendor>` script.

### Builder

`https://github.com/OpenIPC/builder`

Builder is currently defined upstream as a **thin per-device overlay layer** for specific named consumer cameras. It is layered over a fresh Firmware checkout.

Builder should contain only device-specific deltas such as:

- one device defconfig;
- first-boot/customizer policy;
- per-device exclude/prune lists;
- occasional board/sensor files that are genuinely specific to that physical model;
- temporary builder-local package material only when it has not yet been promoted to the proper shared repository.

Common packages, generic SoC support, kernels and toolchains do not belong in Builder. Upstream Builder documentation explicitly says common material belongs in Firmware, while Firmware itself redirects kernel code further down to `OpenIPC/linux`.

For this project that means Builder must remain the last assembly layer. It must not become a duplicate home for FH8626 kernel patches, Divinus/Majestic implementation source, generic media HAL code or general Fullhan runtime support.

### Divinus and streamer code

FH8626-specific Divinus implementation belongs in the Divinus source repository, not as a giant Firmware or Builder patch once the implementation is ready to be maintained there.

Camera-level facts remain in this camera authority repository. Divinus should consume those contracts rather than becoming the only place where board/media knowledge exists.

Majestic is treated by current Firmware rules as a separately maintained streamer. Firmware-side shims that hide streamer defects are explicitly discouraged; the underlying streamer issue should be handled with Majestic maintainers.

### Probing and bring-up tools

OpenIPC Firmware redirects diagnostic, probing, capture and register-poking utilities to:

`https://github.com/OpenIPC/ipctool`

Do not leave such tools as unselected dead source in Firmware merely because they were useful during bring-up.

### Documentation

Current OpenIPC Firmware guidance redirects user/device documentation and how-tos to the OpenIPC Wiki/docs rather than embedding large device manuals into unrelated implementation repositories.

Primary documentation locations:

- `https://github.com/OpenIPC/wiki`
- `https://docs.openipc.org/`

## U-Boot organization

OpenIPC currently does **not** use one universal U-Boot source repository for every supported SoC family.

As checked on 2026-09-17, the OpenIPC organization contains multiple SoC/family-specific U-Boot repositories, including examples such as:

- `https://github.com/OpenIPC/u-boot-xmedia`
- `https://github.com/OpenIPC/u-boot-gk7205v200`
- `https://github.com/OpenIPC/u-boot-hi3516ev200`
- additional SoC-specific `u-boot-*` repositories.

OpenIPC build/distribution tooling also consumes U-Boot as published release artifacts. Current Builder code downloads a `u-boot-<soc>-universal.bin` artifact from OpenIPC Firmware releases for applicable flows, and Firmware image assembly also consumes U-Boot release artifacts.

Therefore there is no current upstream rule that FH8626V100 U-Boot source must be placed inside `OpenIPC/firmware` or `OpenIPC/builder`.

For this project:

- `ArthurKoba/u-boot-fullhan` remains the implementation authority for the working FH8626V100 open U-Boot port until an upstream destination is deliberately chosen;
- do not move U-Boot source into Firmware or Builder merely to make it appear integrated;
- before preparing a U-Boot contribution, re-check the OpenIPC organization for current U-Boot repository structure and determine whether maintainers want a new Fullhan/FH8626 repository, inclusion in an existing Fullhan/U-Boot repository if one now exists, or another arrangement;
- U-Boot release artifact naming/packaging should be aligned with current OpenIPC tooling only after the target upstream ownership is known.

OpenIPC Wiki currently treats both vendor U-Boot and OpenIPC-provided U-Boot images as installation/recovery surfaces and warns that bootloader replacement is higher risk than ordinary firmware replacement. Preserve recovery and flash-safety discipline independently of repository layout.

## Firmware hard gates that matter to FH8626

Current Firmware review standards/compliance rules include the following constraints. Recheck the live files before every contribution because this list can change.

### No factory blobs without a defensible source chain

Do not add `.ko`, `.so`, `.bin` or firmware images extracted from a factory camera when they cannot be traced to a vendor SDK release or buildable source tree. A provenance note alone does not make an unrebuildable binary acceptable upstream.

This is directly relevant to the current FH8626 preservation snapshot containing proprietary Fullhan modules/libraries. Their engineering usefulness does not automatically make them acceptable OpenIPC upstream content.

### No `LD_PRELOAD` shims

Current Firmware rules reject `LD_PRELOAD` in shipped scripts/package/overlay material. Do not solve FH8626 compatibility by globally interposing symbols through a preload shim.

### No runtime patching of vendor blob memory

Do not patch another module's data/code at runtime, derive hook targets from `kallsyms`, or depend on fixed private offsets inside one blob build.

### Stable and reviewable source provenance

New package `*_SITE` values should point to an OpenIPC organization repository or a documented upstream project, not a contributor's personal fork. New source pins should be specific and reviewable; do not loosen an existing commit pin into a moving branch.

This means our personal forks are valid engineering/work repositories, but Firmware must not permanently fetch production packages from them when preparing upstream integration.

### No dead/unselected implementation

New package/source code must actually be wired into Kconfig/Buildroot and selected by at least one intended defconfig, or be explicitly classified by upstream mechanisms when intentionally not built. Unreferenced implementation dumps are not an acceptable preservation strategy in Firmware.

### Hardware evidence for behavior-changing contributions

OpenIPC explicitly distinguishes CI/build proof from target proof. A change that can alter camera behavior normally requires actual hardware evidence: affected board, observable symptom, and before/after logs/measurements/stream behavior as appropriate.

For this project, do not convert host tests, successful compilation or static reverse into an upstream hardware claim.

## Contribution workflow

The general OpenIPC contribution documentation describes the normal external-contributor flow as:

1. fork the relevant OpenIPC repository;
2. make changes in the fork;
3. submit a pull request to the upstream repository;
4. alternatively, prepare/send patch files when GitHub PR workflow cannot be used.

Our project has an additional local rule: automated agents do not create the final upstream pull request unless the repository owner explicitly changes that policy. Agents prepare the clean contribution branch/series, build/test evidence and PR-ready description; the owner performs the final upstream submission.

Before preparing any PR-ready series:

- verify the current upstream base branch;
- re-read that repository's `README`, `AGENTS.md` / `CLAUDE.md`, PR template, contribution rules and review gates;
- remove preservation-only/generated/debug material;
- split unrelated responsibilities by owning repository;
- keep commits coherent and reviewable;
- provide only evidence actually observed;
- do not claim hardware testing that did not occur.

## Current FH8626 implications

The repository cleanup plan should use the following routing until live upstream rules say otherwise:

| Material | Current intended owner |
|---|---|
| FH8626 kernel source/platform/driver changes | `OpenIPC/linux` contribution path |
| Kernel config selection used by OpenIPC build | `OpenIPC/firmware` where family/board config belongs |
| Generic FH8626 shared packages/load policy | `OpenIPC/firmware` if they satisfy provenance/rebuildability rules |
| AJL33PQ0866-only config/GPIO/bootstrap/excludes | `OpenIPC/builder` |
| Divinus FH8626 HAL/media implementation | Divinus upstream/source repository |
| Majestic behavior bug | Majestic maintainers, not a Firmware shim |
| Camera hardware/media contracts | this repository |
| Reverse working state | canonical Ghidra MCP project |
| Bring-up/probing utilities intended for OpenIPC | `OpenIPC/ipctool` where appropriate |
| FH8626 U-Boot source | `ArthurKoba/u-boot-fullhan` until upstream ownership is explicitly chosen |

## Live sources

These links are part of the rule. Agents must open the relevant ones again before implementation/contribution work instead of relying only on this cached summary.

### Firmware ownership and review rules

- `https://github.com/OpenIPC/firmware`
- `https://github.com/OpenIPC/firmware/blob/master/README.md`
- `https://github.com/OpenIPC/firmware/blob/master/CLAUDE.md`
- `https://github.com/OpenIPC/firmware/blob/master/best_practices.md`
- `https://github.com/OpenIPC/firmware/blob/master/pr_compliance_checklist.yaml`

### Builder

- `https://github.com/OpenIPC/builder`
- `https://github.com/OpenIPC/builder/blob/master/README.md`
- `https://github.com/OpenIPC/builder/blob/master/CLAUDE.md`
- `https://github.com/OpenIPC/builder/blob/master/builder.sh`

### Linux

- `https://github.com/OpenIPC/linux`
- `https://github.com/OpenIPC/linux/blob/openipc/README.md`

### General OpenIPC contribution/development documentation

- `https://github.com/OpenIPC/wiki/blob/master/ru/contribute.md`
- `https://github.com/OpenIPC/wiki/blob/master/en/source-code.md`
- `https://github.com/OpenIPC/docs`

### U-Boot / recovery / current organization

- `https://github.com/OpenIPC`
- `https://github.com/OpenIPC/wiki/blob/master/en/help-uboot.md`
- `https://github.com/OpenIPC/wiki/blob/master/en/installation.md`
- `https://github.com/OpenIPC/firmware/blob/master/.github/workflows/image.yml`
- `https://github.com/OpenIPC/u-boot-xmedia`
- `https://github.com/OpenIPC/u-boot-gk7205v200`
- `https://github.com/OpenIPC/u-boot-hi3516ev200`

## Maintenance rule

This file should carry a current `Checked against live OpenIPC sources` date.

Whenever a future agent discovers that OpenIPC changed repository ownership, branch conventions, contribution requirements, binary/provenance policy, build gates or U-Boot organization, that agent must update this document and any affected local roadmap/ownership docs before relying on the changed rule.
