# FH8626V100 U-Boot port findings

Checked: `2026-09-17`.

Implementation source authority is `ArthurKoba/u-boot-fullhan`, branch `fh8626v100-mainline`. The observed audit tip is `ae63365e10b38e5b9ed3bd5173a0ed3e5c8f9996`; current repository/base refs are recorded in `docs/process/upstream-integration.md`.

## Current conclusion

The current port is a hardware-used, near-production U-Boot implementation for the **ANJIA AJL33PQ0866**. The source/architecture audit did not identify a missing bootloader capability required for the currently exercised boot/recovery path.

However, the currently hardware-proven flash layout is a **stock-compatible migration layout**, not the intended final OpenIPC layout. The project goal is to migrate away from the factory partition arrangement and converge on the current OpenIPC NOR conventions wherever the final FH8626 kernel/rootfs sizes permit it.

Do not call the present artifact a universal FH8626V100 bootloader. The exercised implementation contains board-specific DDR/bootstrap data, GPIO policy, PHY wiring and an ANJIA-specific Boot ROM manifest. A matching SoC name alone is not enough to establish persistent-flash compatibility with another board.

## Accepted stock-compatible hardware state

`HARDWARE_PASS`: the port was first chainloaded non-persistently from stock U-Boot and then migrated into the board's original U-Boot partition.

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

The already-proven migration procedure writes only the stock 192 KiB U-Boot partition at `0x20000`; it preserves the stock 64 KiB bootstrap at `0x00000` and environment at `0x10000`. This remains a safe compatibility/recovery reference, not the target product flash map.

## Production size contract

The currently validated Fullhan Boot-ROM-visible U-Boot descriptor is:

- load/entry address: `0xA0800000`;
- raw descriptor size: `0x2BAE4`;
- aligned envelope: `0x2BB00`;
- JAMCRC: `0x251D4C31`.

The audited CI build at `ae63365e...` produced a raw U-Boot size of `0x2B7C8`, leaving only `0x334` bytes, or 820 bytes, inside that envelope.

The 192 KiB physical U-Boot slot is therefore large enough, but the ROM-visible envelope is tight. The compact flash target deliberately omits several non-essential development features, including MMC/FAT commands, `tftpput`, long help, line editing/completion and optional TFTP tuning variables. The RAM recovery target retains a larger development feature set.

Do not add bootcount/failsafe, extra filesystems or other convenience features to the production target without a measured requirement and size budget.

## OpenIPC-relevant environment contract

The current production target already provides the useful OpenIPC-facing contracts:

- `soc=fh8626v100`;
- `baseaddr=0xa1000000`;
- `flashsize=0x800000`;
- SPI environment support;
- `bootcmdnor` / NOR boot path;
- `sf`, `bootm`, environment, memory/CRC, MII, ping and TFTP support;
- `ethaddr=${ethaddr}` propagated into Linux bootargs.

The `ethaddr` bootarg is intentional. The current FH8626 Linux platform consumes `early_param("ethaddr", ...)`, validates it and passes a usable value to Fullhan GMAC platform data. Do not drop that change as an apparently redundant WIP fix.

The current `kload`, vendor-compatible `gpio <pin> out <0|1>`, fixed ANJIA `bootfile`, and stock `set_gpio` sequence are migration/compatibility behavior. They need not define the final OpenIPC-native default environment once the stock environment is no longer preserved.

## Target OpenIPC-native 8 MiB layout

Current OpenIPC NOR guidance uses the 8 MiB layout:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

with kernel at `0x50000` and rootfs at `0x250000`.

FH8626 can potentially match those boundaries cleanly. The Fullhan first stage needs a 64 KiB Boot ROM container, while the current U-Boot occupies a 192 KiB physical slot. Those two pieces total exactly 256 KiB.

The target geometry is therefore:

| Region | Offset | Size | Intended content |
|---|---:|---:|---|
| boot | `0x000000` | 256 KiB | 64 KiB Fullhan Boot ROM container + 192 KiB U-Boot |
| env | `0x040000` | 64 KiB | OpenIPC U-Boot environment |
| kernel | `0x050000` | 2048 KiB | OpenIPC kernel |
| rootfs | `0x250000` | 5120 KiB | OpenIPC SquashFS |
| rootfs_data | `0x750000` | remainder (704 KiB on 8 MiB NOR) | writable overlay |

This layout is not yet hardware-accepted on FH8626. It is the target migration design.

### Required bootloader changes for that layout

The current stock container tells the Fullhan Boot ROM to load U-Boot from flash offset `0x20000`. Because the container is now reconstructed as structured data, that descriptor can be changed to load the same U-Boot image from `0x10000` while keeping its RAM load/entry address at `0xA0800000`.

That produces the desired `boot` region:

- `0x00000..0x0ffff`: Fullhan Boot ROM container / DDR parameters;
- `0x10000..0x3ffff`: 192 KiB U-Boot partition;
- `0x40000..0x4ffff`: 64 KiB OpenIPC environment;
- `0x50000`: kernel start, already matching OpenIPC.

U-Boot must then use environment offset `0x40000` and compile an OpenIPC-native default environment instead of depending on the retained vendor environment.

The descriptor-offset change is technically plausible from the recovered container format but is **not yet `HARDWARE_PASS`**. It requires a deliberately staged cold-boot test with external SPI recovery available.

## Kernel-size gate

The remaining question before declaring the standard OpenIPC 8 MiB layout feasible is the final FH8626 `uImage` size.

The stock Fullhan kernel descriptor is larger than 2 MiB, and the historical FH8626 Firmware preservation snapshot reserved 3 MiB for kernel. Neither fact proves that the final curated OpenIPC kernel needs 3 MiB.

Before preserving any custom partition map:

1. build the final intended `openipc-linux/fullhan-fh8626v100` kernel with the target Firmware config;
2. measure the produced `uImage` exactly;
3. if it fits within 2 MiB, use the standard OpenIPC 8 MiB kernel/rootfs boundaries;
4. if it does not fit, first review kernel configuration/compression/built-in choices and reduce it where reasonable;
5. use a SoC-specific OpenIPC layout only if the final required kernel genuinely cannot fit the standard 2 MiB partition.

A historical 3 MiB reservation is not sufficient justification for a permanent custom OpenIPC layout.

## Boot ROM container

`board/fullhan/fh8626v100/bootrom.json` is a structured reconstruction of the exercised ANJIA bootstrap container. It contains board metadata (`ANJIA`, `AJL33PQ0866`) and recovered DDR/Boot-ROM parameter records. The build generates the 64 KiB container deterministically; no executable vendor U-Boot binary is embedded in the source repository.

The reconstruction is also what makes an OpenIPC-native first-320-KiB migration possible: the U-Boot descriptor is no longer an opaque stock blob that must remain at its original flash offset.

Its provenance must still be explicit before external contribution: identify the exact retained source dump/evidence object, explain the extraction/reconstruction method, state that the JSON contains recovered parameter/data records rather than a copied executable blob, and preserve independent hardware validation. Do not infer licensing status merely from the current SPDX line; OpenIPC maintainers must be given enough provenance to decide the acceptable form.

## Driver/source audit

### Strong/appropriate pieces

- FH8626 architecture/board support is based on modern upstream U-Boot rather than copied Fullhan U-Boot 2010 source.
- UART/GPIO/SPI/Ethernet/timer use current U-Boot driver-model/devicetree integration where practical.
- The dedicated FH8626 DWMAC glue is appropriate for the SoC-specific PMU, RMII speed and reset contract.
- The unusual Ethernet reset sequence that writes `~RESET_MASK` and waits for `0xffffffff` matches the retained FH8626 Linux PMU implementation; it is not an audit-found bug.
- The DesignWare SPI Fullhan wrapper/FIFO handling addresses a hardware-proven receive-overrun problem on large NOR reads.
- The RAM target gives a non-persistent validation/recovery route before flash writes.
- Boot-container tooling has host tests and branch CI builds both flash and RAM targets and checks output geometry/checksums.

### Contribution-cleanup items

These are review/ownership improvements rather than target failures.

1. **Separate SoC-generic and AJL33PQ0866-specific identity.** Split common FH8626 support from ANJIA board data/policy.
2. **Do not publish the current stock-compatible binary as `universal`.** Other FH8626 boards need independent DDR/bootstrap/PHY/GPIO/RAM validation.
3. **Replace stock-layout defaults with OpenIPC-native defaults in the final product target.** Keep stock compatibility as a migration/recovery target if useful.
4. **Fold the top WIP commit into the clean series.** `ethaddr` is required behavior and should land in the logical board/kernel-compatibility change.
5. **Keep common-driver prerequisites reviewable.** DesignWare MDIO clock-range support is a generic prerequisite; Fullhan SPI quirks and FH-specific glue should be isolated.
6. **Revisit FH-specific vendor-GPIO compatibility in generic `cmd/gpio.c`.** It can remain in a stock-migration build but should not be required by the final OpenIPC environment.
7. **Add contribution gates.** Run current U-Boot style/checkpatch checks over the curated patches in addition to existing CI.
8. **Document Boot ROM reconstruction provenance explicitly.** Preserve the evidence chain separately from generated artifacts.

## Feature completeness for the OpenIPC target

No current hard requirement was found for vendor-only commands such as `upgrade`, `fastbootcmd`, `arc_go` or `chpart`.

Likewise, `tftpput`, MMC/FAT recovery and a richer interactive shell are useful development/recovery conveniences but are not blockers because TFTP download and SPI operations are available, a separate RAM recovery target exists, and external SPI recovery remains available during bootloader migration.

Automatic bootcount/fallback remains optional until a concrete OpenIPC recovery policy and size budget are defined.

## OpenIPC integration path

OpenIPC currently maintains multiple SoC/family-specific `u-boot-*` repositories rather than one universal bootloader-source repository. No current OpenIPC Fullhan U-Boot source repository was identified during the 2026-09-17 check.

The intended handoff sequence is:

1. keep `ArthurKoba/u-boot-fullhan` as source authority while curating the branch;
2. create a clean OpenIPC-native board target while preserving the already-proven stock-compatible target as migration evidence/reference;
3. validate the OpenIPC-native first-320-KiB geometry and default environment on hardware;
4. measure the final FH8626 kernel against the standard OpenIPC 2 MiB kernel budget and use the standard 8 MiB map if it fits;
5. prepare the OpenIPC-facing clean patch series from a verified current U-Boot base;
6. present OpenIPC maintainers with source, hardware evidence, Boot ROM provenance and proposed artifact scheme and ask which OpenIPC repository should own the Fullhan U-Boot source;
7. once ownership/naming is agreed, integrate the resulting artifact into Firmware image assembly;
8. keep Builder limited to final per-device selection/configuration rather than U-Boot source.

A reasonable contribution series to propose is:

1. generic DesignWare prerequisite(s);
2. `arm: fullhan: add FH8626V100 SoC support`;
3. FH8626 Ethernet/SPI platform integration;
4. `board: fullhan: add ANJIA AJL33PQ0866`;
5. Fullhan Boot ROM container/tooling and tests;
6. OpenIPC-native layout/default environment plus stock-migration support;
7. CI/documentation and migration safety.

The exact split must be adjusted to destination-repository rules before submission.

## Migration safety boundary

Moving from the already-proven stock-compatible geometry to the OpenIPC-native geometry is a one-time destructive boot-layout migration.

The final design moves U-Boot storage from `0x20000` to `0x10000` and moves environment from `0x10000` to `0x40000`. Therefore it cannot preserve the old environment/U-Boot pair in place during the conversion.

Do the first hardware conversion only from the RAM U-Boot or an external programmer with a verified complete flash backup and external recovery available. Write and read-back-verify all new boot components before resetting. Power loss during the conversion can require external SPI recovery.

After the OpenIPC-native layout is accepted, later updates should use ordinary OpenIPC partition/update semantics rather than repeating stock migration logic.

## Mainline U-Boot boundary

Passing the work to OpenIPC and upstreaming it to Das U-Boot are separate goals.

If mainline U-Boot submission is pursued later, rebase/curate against the then-current U-Boot `master`/`next`, follow current coding style, run `scripts/checkpatch.pl`/`b4 prep --check`, keep one logical change per patch, preserve `Signed-off-by` provenance, and send the series using the current U-Boot mailing-list process.

Do not force OpenIPC integration to wait for mainline U-Boot acceptance unless OpenIPC maintainers explicitly require that.

## Live references to recheck before contribution

- OpenIPC current NOR/U-Boot layout notes: `https://github.com/OpenIPC/wiki/blob/master/en/help-uboot.md`
- OpenIPC sysupgrade notes: `https://github.com/OpenIPC/wiki/blob/master/en/sysupgrade.md`
- OpenIPC Firmware image assembly: `https://github.com/OpenIPC/firmware/blob/master/.github/workflows/image.yml`
- OpenIPC organization/repositories: `https://github.com/OpenIPC`
- Das U-Boot patch process: `https://docs.u-boot.org/en/latest/develop/sending_patches.html`
- Das U-Boot coding style: `https://docs.u-boot.org/en/latest/develop/codingstyle.html`

These links are live authority for their respective projects. Recheck them before actual handoff; update this document if ownership/layout/contribution rules change.
