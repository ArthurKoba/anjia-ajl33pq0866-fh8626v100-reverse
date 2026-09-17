# FH8626V100 U-Boot port findings

Checked: `2026-09-17`.

Implementation source authority is `ArthurKoba/u-boot-fullhan`, branch `fh8626v100-mainline`. The observed audit tip is `ae63365e10b38e5b9ed3bd5173a0ed3e5c8f9996`; current repository/base refs are recorded in `docs/process/upstream-integration.md`.

## Current conclusion

The current port is a hardware-used, near-production U-Boot implementation for the **ANJIA AJL33PQ0866**. The source/architecture audit did not identify a missing bootloader capability that is required for the currently exercised OpenIPC boot, recovery and update path.

The primary remaining work is contribution curation and OpenIPC integration, not bootloader reimplementation.

Do not call the present artifact a universal FH8626V100 bootloader. The exercised implementation contains board-specific DDR/bootstrap data, GPIO policy, PHY wiring, SPI geometry and an ANJIA-specific Boot ROM manifest. A matching SoC name alone is not enough to establish persistent-flash compatibility with another board.

## Accepted hardware state

`HARDWARE_PASS`: the port was first chainloaded non-persistently from stock U-Boot and then migrated into the board's persistent U-Boot partition.

The exercised path proves:

- ARM1176 execution on a modern upstream-U-Boot base;
- UART console;
- 64 MiB DRAM discovery/relocation;
- timer/autoboot behavior;
- both FH8626 GPIO banks;
- SPI NOR identification and repeated reads;
- GMAC/MDIO, PHY identification, link negotiation, ping and TFTP;
- legacy image verification and Linux handoff;
- persistent SPI environment loading;
- cold boot through the unchanged stock Fullhan bootstrap;
- boot of OpenIPC and the complete installed stock firmware through the replacement U-Boot.

The persistent migration procedure writes only the 192 KiB U-Boot partition at `0x20000`; it preserves the 64 KiB stock bootstrap at `0x00000` and the 64 KiB environment at `0x10000`. Complete read-back/CRC/compare is required before reset.

## Production size contract

The Fullhan Boot ROM-visible U-Boot descriptor is fixed to:

- load/entry address: `0xA0800000`;
- raw descriptor size: `0x2BAE4`;
- aligned envelope: `0x2BB00`;
- JAMCRC: `0x251D4C31`.

The audited CI build at `ae63365e...` produced a raw U-Boot size of `0x2B7C8`, leaving only `0x334` bytes, or 820 bytes, inside that envelope.

This size limit is a real product constraint. The compact flash target deliberately omits several non-essential development features, including MMC/FAT commands, `tftpput`, long help, line editing/completion and optional TFTP tuning variables. The RAM recovery target retains a larger development feature set.

Do not add bootcount/failsafe, extra filesystems or other convenience features to the production target merely because another OpenIPC U-Boot provides them. First establish an actual requirement and a size budget.

## OpenIPC-relevant environment contract

The current production target provides the OpenIPC-facing variables and commands needed by the exercised flow:

- `soc=fh8626v100`;
- `baseaddr=0xa1000000`;
- `flashsize=0x800000`;
- persistent SPI environment;
- `bootcmdnor=run openipc_boot`;
- `sf`, `bootm`, environment, memory/CRC, MII, ping and TFTP support;
- `kload` compatibility for the retained stock environment;
- vendor-compatible `gpio <pin> out <0|1>` syntax for stock migration;
- `ethaddr=${ethaddr}` propagated into Linux bootargs.

The `ethaddr` bootarg is intentional. The current FH8626 Linux platform consumes `early_param("ethaddr", ...)`, validates it and passes a usable value to the Fullhan GMAC platform data. Do not drop the latest U-Boot `ethaddr` change as an apparently redundant WIP fix.

The fixed default `bootfile=anjia-ajl33pq0866.uImage` and the current `set_gpio` sequence are board-specific convenience/policy and should not be presented as generic FH8626 defaults if the port is generalized.

## Flash layout consistency

The current U-Boot, FH8626 Linux platform and preserved Firmware integration agree on the exercised 8 MiB NOR layout:

| Region | Offset | Size |
|---|---:|---:|
| bootstrap | `0x000000` | 64 KiB |
| uboot-env | `0x010000` | 64 KiB |
| uboot | `0x020000` | 192 KiB |
| kernel | `0x050000` | 3072 KiB |
| rootfs_data | `0x350000` | 1024 KiB |
| rootfs | `0x450000` | 3776 KiB / remainder |

This is internally coherent but differs from the ordinary OpenIPC 8 MiB image examples that use a 2 MiB kernel and place rootfs at `0x250000`. FH8626 integration therefore needs an explicit image-assembly/layout path rather than silently reusing the generic offset scheme.

## Boot ROM container

`board/fullhan/fh8626v100/bootrom.json` is a structured reconstruction of the exercised ANJIA bootstrap container. It contains board metadata (`ANJIA`, `AJL33PQ0866`) and recovered DDR/Boot-ROM parameter records. The build generates the 64 KiB bootstrap deterministically; no executable vendor U-Boot binary is embedded in the source repository.

The reconstruction is valuable and reproducible, but its provenance should be made explicit before external contribution: identify the exact retained source dump/evidence object, explain the extraction/reconstruction method, state that the JSON contains recovered parameter/data records rather than a copied executable blob, and preserve independent hardware validation. Do not infer licensing status merely from the current SPDX line; contribution review should make the provenance clear enough for OpenIPC maintainers to decide the acceptable form.

## Driver/source audit

### Strong/appropriate pieces

- FH8626 architecture/board support is based on modern upstream U-Boot rather than copied Fullhan U-Boot 2010 source.
- UART/GPIO/SPI/Ethernet/timer use current U-Boot driver-model/devicetree integration where practical.
- The dedicated FH8626 DWMAC glue is appropriate for the SoC-specific PMU, RMII speed and reset contract.
- The apparently unusual Ethernet reset sequence that writes `~RESET_MASK` and waits for `0xffffffff` matches the independently retained FH8626 Linux PMU implementation; it is not an audit-found bug.
- The DesignWare SPI Fullhan wrapper/FIFO handling addresses a hardware-proven receive-overrun problem on large NOR reads.
- The RAM target gives a non-persistent validation/recovery route before flash writes.
- Boot-container tooling has host tests and the branch CI builds both flash and RAM targets and checks output geometry/checksums.

### Contribution-cleanup items

These are not target failures; they are review/ownership improvements before handing the work to OpenIPC or mainline U-Boot.

1. **Separate SoC-generic and AJL33PQ0866-specific identity.** The current `fh8626v100` target/DTS is called a reference platform but contains exercised board wiring and policy. Prefer an explicit board target/artifact, or split common FH8626 SoC data from an `anjia-ajl33pq0866` board DTS/config.
2. **Do not publish the current binary as `universal`.** Other FH8626 boards must independently match DDR/bootstrap, PHY, GPIO, RAM and flash geometry before persistent use.
3. **Fold the top WIP commit into the clean series.** `ethaddr` is required behavior and should land in the logical board/kernel-compatibility change, not remain as `WIP: preserve latest ...`.
4. **Keep common-driver prerequisites reviewable.** The DesignWare MDIO clock-range support is a clean generic prerequisite; Fullhan SPI quirks and FH-specific glue should be isolated enough that reviewers can reason about their blast radius.
5. **Revisit the FH-specific compatibility path in generic `cmd/gpio.c`.** It is justified for preserving the stock environment but is an upstream-U-Boot cleanliness concern. For an OpenIPC-owned fork it may remain pragmatic; for mainline U-Boot prefer a board-local compatibility mechanism or migration away from the vendor syntax.
6. **Add contribution gates.** The existing CI proves builds/tests/artifact geometry, but a curated series should also run current U-Boot style/checkpatch checks over the submitted patches.
7. **Document Boot ROM reconstruction provenance explicitly.** Preserve the evidence chain separately from generated artifacts.

## Feature completeness for the current OpenIPC path

No current hard requirement was found for the production target to provide vendor-only commands such as `upgrade`, `fastbootcmd`, `arc_go` or `chpart`.

Likewise, `tftpput`, MMC/FAT recovery and a richer interactive shell are useful development/recovery conveniences but are not blockers for the exercised OpenIPC deployment because:

- TFTP download is present;
- SPI read/write/erase are present;
- a dedicated RAM U-Boot target provides richer recovery tooling;
- an external SPI programmer remains the recovery authority for destructive bootloader migration;
- the production image has only 820 bytes of current ROM-envelope headroom.

Automatic bootcount/fallback is a possible future reliability feature, not an audit-established prerequisite. It should be considered only with a defined failure/recovery policy and measured size impact.

## OpenIPC integration path

OpenIPC currently maintains multiple SoC/family-specific `u-boot-*` repositories rather than one universal bootloader-source repository. No current OpenIPC Fullhan U-Boot source repository was identified during the 2026-09-17 check.

The recommended handoff sequence is therefore:

1. keep `ArthurKoba/u-boot-fullhan` as the source authority while curating the branch;
2. prepare an OpenIPC-facing clean series from a verified current U-Boot base without changing hardware-proven behavior;
3. present OpenIPC maintainers with the source repository, hardware evidence, boot-chain/layout documentation and proposed artifact scheme, and ask which OpenIPC repository ownership they want;
4. if OpenIPC chooses a new Fullhan repository, transfer/import the curated source there rather than copying U-Boot source into Firmware or Builder;
5. publish board-specific boot artifacts according to the naming convention agreed with maintainers;
6. add the FH8626 image-assembly path in OpenIPC Firmware only after U-Boot ownership and artifact naming are settled;
7. keep Builder limited to final per-device selection/configuration rather than U-Boot source.

A reasonable contribution series to propose for review is:

1. generic DesignWare prerequisite(s);
2. `arm: fullhan: add FH8626V100 SoC support`;
3. FH8626 Ethernet/SPI platform integration;
4. `board: fullhan: add ANJIA AJL33PQ0866`;
5. FH8626 boot-container/recovery tooling and tests;
6. CI/documentation for produced artifacts and migration safety.

The exact split should be adjusted to the destination repository's current maintainers/rules before submission.

## OpenIPC image assembly implication

Current OpenIPC Firmware image assembly downloads U-Boot artifacts from the Firmware `latest` release and has both generic and special board/DDR-specific assembly paths. That makes an FH8626-specific path structurally reasonable.

For the exercised AJL33PQ0866 layout, a full 8 MiB image assembler must preserve the first 320 KiB boot region, write the kernel at 320 KiB (`0x50000`), preserve the 1 MiB `rootfs_data` region beginning at `0x350000`, and write rootfs at 4416 KiB (`0x450000`). Do not reuse the ordinary 2368 KiB rootfs offset.

Possible later integration with OpenIPC `defib` is useful but not a prerequisite for accepting the U-Boot source. `defib` already has a plugin-oriented recovery architecture and can use custom U-Boot binaries; FH8626 Boot-ROM/recovery support can be considered as a separate follow-up.

## Mainline U-Boot boundary

Passing the work to OpenIPC and upstreaming it to Das U-Boot are separate goals.

If mainline U-Boot submission is pursued later, rebase/curate against the then-current U-Boot `master`/`next`, follow current coding style, run `scripts/checkpatch.pl`/`b4 prep --check`, keep one logical change per patch, preserve `Signed-off-by` provenance, and send the series using the current U-Boot mailing-list process.

Do not force OpenIPC integration to wait for mainline U-Boot acceptance unless OpenIPC maintainers explicitly require that.

## Live references to recheck before contribution

- OpenIPC Firmware image assembly: `https://github.com/OpenIPC/firmware/blob/master/.github/workflows/image.yml`
- OpenIPC U-Boot/recovery notes: `https://github.com/OpenIPC/wiki/blob/master/en/help-uboot.md`
- OpenIPC sysupgrade notes: `https://github.com/OpenIPC/wiki/blob/master/en/sysupgrade.md`
- OpenIPC organization/repositories: `https://github.com/OpenIPC`
- Das U-Boot patch process: `https://docs.u-boot.org/en/latest/develop/sending_patches.html`
- Das U-Boot coding style: `https://docs.u-boot.org/en/latest/develop/codingstyle.html`

These links are live authority for their respective projects. Recheck them before the actual handoff; update this document if the ownership/layout/contribution rules have changed.
