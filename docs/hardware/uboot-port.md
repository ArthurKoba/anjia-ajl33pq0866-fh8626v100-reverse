# FH8626V100 U-Boot port findings

Checked: `2026-09-17`.

Implementation source authority is `ArthurKoba/u-boot-fullhan`.

Current refs:

- hardware-proven stock-compatible baseline: `fh8626v100-mainline@49fe46e9ddb786e232d1359f9cee68c914a3a8db`;
- implemented OpenIPC-native candidate: `fh8626v100-openipc-native@99c477674acc250b3177e3ae6eaaa1680c28c809`.

The baseline remains the recovery authority until the native candidate passes a complete cold-boot migration test.

## Current conclusion

The underlying FH8626V100 U-Boot port is hardware-proven on ANJIA AJL33PQ0866 and does not need reimplementation. The project has now implemented the intended architectural transition away from the factory Fullhan flash/environment contract toward a native OpenIPC 8 MiB NOR contract.

The OpenIPC-native code is a `SOURCE/BUILD CANDIDATE`, not `HARDWARE_PASS` yet.

The product goal is explicit:

- use factory Fullhan data only as board bring-up/migration/recovery evidence;
- use OpenIPC partition/environment/update conventions as the normal installed state;
- keep reusable FH8626 SoC support separate from AJL33PQ0866 board-specific Boot-ROM/DDR/GPIO/PHY data;
- do not carry factory commands or partition policy into production merely for compatibility.

## Hardware-proven baseline

`HARDWARE_PASS` on the stock-compatible baseline includes:

- modern upstream-U-Boot ARM1176 execution;
- UART console;
- 64 MiB DRAM discovery and relocation;
- timer/autoboot behavior;
- both FH8626 GPIO banks;
- SPI NOR detection and repeated reads at 50 MHz;
- legacy image verification and Linux handoff;
- FH8626 GMAC/MDIO, PHY detection, 100 Mbit/s link, ping and TFTP;
- persistent SPI environment access;
- cold boot through the Fullhan Boot ROM data container;
- successful OpenIPC boot through the replacement U-Boot;
- successful boot of the complete installed factory firmware through the replacement U-Boot.

The proven baseline used the factory-compatible placement:

- Boot ROM container `0x00000..0x0ffff`;
- environment `0x10000..0x1ffff`;
- U-Boot `0x20000..0x4ffff`;
- kernel at `0x50000`.

That geometry is now a recovery/reference state, not the target architecture.

## Implemented OpenIPC-native layout

The candidate uses the standard OpenIPC 8 MiB geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

with these offsets:

| Region | Offset | Size |
|---|---:|---:|
| boot | `0x000000` | 256 KiB |
| env | `0x040000` | 64 KiB |
| kernel | `0x050000` | 2048 KiB |
| rootfs | `0x250000` | 5120 KiB |
| rootfs_data | `0x750000` | 704 KiB / remainder |

FH8626V100 has only an internal board-specific split inside the normal 256 KiB `boot` partition:

- `0x00000..0x0ffff`: reconstructed Fullhan Boot ROM data container;
- `0x10000..0x3ffff`: fixed 192 KiB U-Boot slot.

The reconstructed U-Boot descriptor is changed for the native layout to:

- flash source offset `0x10000`;
- raw/aligned ROM-visible U-Boot size `0x30000`;
- RAM load address `0xA0800000`;
- entry address `0xA0800000`;
- fixed JAMCRC `0x251D4C31`, satisfied by a four-byte correction in the padded U-Boot slot.

The RAM load/entry contract remains unchanged; only the NOR source location and envelope used by the generated board container are changed.

## OpenIPC-native production environment

The production target now uses:

- `CONFIG_ENV_OFFSET=0x40000`;
- prompt `OpenIPC # `;
- `soc=fh8626v100`;
- `manufacturer=fullhan`;
- `baseaddr=0xa1000000`;
- `flashsize=0x800000`;
- `osmem=39M` and `totalmem=64M`;
- standard OpenIPC-style `mtdpartsnor8m` and `setnor8m`;
- `bootcmdnor` reading the 2 MiB kernel partition from `0x50000`;
- `uknor8m` using `uImage.${soc}`;
- `urnor8m` using `rootfs.squashfs.${soc}` at `0x250000`;
- Linux root `/dev/mtdblock3`;
- `ethaddr=${ethaddr}` propagated into Linux bootargs.

The `ethaddr` propagation remains required by the current FH8626 Linux platform, which consumes the boot argument and passes a usable MAC into Fullhan GMAC platform data.

The production target no longer requires:

- factory `kload`;
- `ethact=FH EMAC` cleanup;
- factory `gpio <pin> out <0|1>` syntax;
- stock `bootcmd`/`set_gpio` environment policy.

Those factory compatibility helpers are now behind `CONFIG_FH8626V100_STOCK_COMPAT` and enabled only in the RAM migration/recovery configuration.

## Production feature budget

The native candidate no longer inherits the old `0x2bb00` factory U-Boot envelope. It uses the complete 192 KiB slot inside the OpenIPC 256 KiB boot partition.

The candidate restores useful recovery/operator features in production:

- `tftpput`;
- TFTP tuning variables;
- command-line editing;
- autocomplete;
- long help;
- `sleep`;
- existing `sf`, `bootm`, memory/CRC, GPIO, MII, ping and TFTP download commands.

Observed build result for `fh8626v100-openipc-native@99c47767...`:

- raw production `u-boot.bin`: `0x2ed18` bytes;
- padded U-Boot slot: `0x30000` bytes;
- CRC-fixup reserve: 4 bytes;
- remaining payload headroom: 4836 bytes.

This is a much healthier production budget than the hardware-proven stock-compatible build, which had only 820 bytes remaining under the old ROM-visible envelope.

MMC/FAT is still not enabled merely for feature parity: the present U-Boot path has no accepted need for SD-card boot/recovery, and adding a new controller/filesystem path should follow a concrete recovery requirement rather than checkbox parity.

Automatic bootcount/fallback likewise remains a future reliability feature until a concrete OpenIPC policy/storage mechanism is defined.

## Native artifact tooling

The candidate adds `tools/fh8626_openipc_boot.py` and matching source tests.

The packer emits board-specific artifacts:

- `u-boot-fh8626v100-anjia-ajl33pq0866.bin` — exact 256 KiB OpenIPC `boot` partition;
- `...-bootstrap.bin` — 64 KiB reconstructed Boot ROM container;
- `...-uboot.bin` — 192 KiB padded U-Boot slot;
- `...-raw.bin` — raw linked production U-Boot;
- `...-ram.bin` — non-persistent migration/recovery U-Boot;
- `SHA256SUMS`.

The older `tools/fh8626_bootchain.py` remains the stock-container parsing/reconstruction oracle. It does not define the final partition map.

The native packer validates:

- exact 256 KiB boot artifact size;
- U-Boot descriptor at flash offset `0x10000`;
- `0x30000` raw/aligned descriptor sizes;
- load/entry `0xA0800000`;
- descriptor/payload JAMCRC agreement;
- standard OpenIPC env/kernel/rootfs/rootfs_data offsets in its tests.

## Board versus SoC boundary

The candidate makes the target identity explicit:

- SoC: Fullhan FH8626V100;
- validated board: ANJIA AJL33PQ0866;
- DTS model: `ANJIA AJL33PQ0866`;
- compatibles: `anjia,ajl33pq0866`, `fullhan,fh8626v100`.

Do not publish the current artifact as `fh8626v100-universal`. The board-specific container includes ANJIA metadata and recovered DDR parameters, while RMII pad/reset wiring and persistent migration assumptions are also board-specific.

A future second FH8626 board should reuse SoC drivers/support but provide and validate its own board data/profile where required.

## Kernel-size gate

The only architectural question still capable of blocking the standard OpenIPC 8 MiB map is the final OpenIPC kernel size.

The target is a 2 MiB `uImage` partition, not the historical 3 MiB reservation.

Required order:

1. build the intended `openipc-linux/fullhan-fh8626v100` kernel with the final relevant config;
2. measure `uImage` exactly;
3. if it fits 2 MiB, keep the standard OpenIPC layout;
4. if it does not fit, first review compression/config/built-in/module choices;
5. accept a nonstandard layout only if the required kernel genuinely cannot fit after reasonable cleanup.

The old Firmware snapshot is not evidence that 3 MiB is required.

## Migration safety

The native layout transition is a one-time destructive conversion. It moves U-Boot from `0x20000` to `0x10000`, discards the factory environment at `0x10000`, creates OpenIPC environment at `0x40000`, and relocates rootfs to `0x250000`.

The candidate repository documents a staged procedure in `doc/board/fullhan/fh8626v100-openipc-migration.rst`:

1. enter RAM recovery U-Boot;
2. verify SPI/network before writes;
3. write/read-back-compare final kernel;
4. write/read-back-compare final rootfs;
5. erase the new rootfs_data region;
6. write the 256 KiB native boot partition last and compare every byte;
7. erase the new environment sector;
8. reset only after all comparisons pass.

The first test requires a verified full-flash backup, UART and an external SPI programmer. Power loss after destructive writes can require external recovery.

The native layout is not `HARDWARE_PASS` until the physical camera cold-boots the complete migrated image and the new MTD/environment/runtime behavior is verified.

## Build/CI evidence and current workflow limitation

The repository's existing auto-workflow successfully compiled both the OpenIPC-native production target and the RAM migration target and executed the native artifact packer. The observed production size above comes from that build.

The workflow then failed only because its final check still looks for the removed stock-layout artifact names (`u-boot-fh8626v100-partition.bin`, `...-nor.bin`). The bridge GitHub App is explicitly denied permission to modify `.github/workflows/build.yml` without GitHub `workflows` permission.

Therefore:

- do not classify the current red workflow as a compiler failure;
- do not generate fake/obsolete stock artifacts merely to satisfy the stale check;
- update the workflow to native board-specific artifact names using an identity with workflow-write permission before contribution.

## Contribution curation

The current native branch is an engineering/hardware-test candidate, not yet the final upstream patch series.

After hardware acceptance:

- squash/reorder implementation-history cleanup such as the executable-mode fix;
- fold the historical `ethaddr` WIP into its logical board/Linux compatibility patch;
- keep generic DesignWare prerequisites independently reviewable;
- keep Fullhan SPI/Ethernet quirks isolated from board policy;
- preserve `Signed-off-by` provenance in the final series;
- run current Das U-Boot checkpatch/style gates;
- document Boot ROM reconstruction provenance clearly;
- recheck current OpenIPC repository ownership before submission.

OpenIPC currently uses multiple SoC/family U-Boot repositories rather than a single universal source tree. The FH8626 source should remain in `ArthurKoba/u-boot-fullhan` until OpenIPC maintainers choose the intended organization-level destination. Do not copy U-Boot source into Firmware or Builder.

## OpenIPC integration path

After native hardware acceptance and repository-ownership agreement:

1. publish/consume the board-specific 256 KiB boot artifact using the naming convention agreed with OpenIPC;
2. use normal OpenIPC 8 MiB kernel/rootfs offsets if the 2 MiB kernel gate passes;
3. update Firmware image assembly rather than carrying a special historical FH8626 layout;
4. keep Builder as the final thin device-selection/configuration layer;
5. treat `defib` support as optional later recovery integration.

## Live references to recheck before contribution

- OpenIPC U-Boot/layout notes: `https://github.com/OpenIPC/wiki/blob/master/en/help-uboot.md`
- OpenIPC sysupgrade notes: `https://github.com/OpenIPC/wiki/blob/master/en/sysupgrade.md`
- OpenIPC Firmware image assembly: `https://github.com/OpenIPC/firmware/blob/master/.github/workflows/image.yml`
- OpenIPC organization: `https://github.com/OpenIPC`
- Das U-Boot patch process: `https://docs.u-boot.org/en/latest/develop/sending_patches.html`
- Das U-Boot coding style: `https://docs.u-boot.org/en/latest/develop/codingstyle.html`

These are live upstream authorities. Recheck them before the actual handoff and update this document if upstream rules or ownership change.
