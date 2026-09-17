# FH8626V100 U-Boot port findings

Checked: `2026-09-17`.

Implementation source authority is `ArthurKoba/u-boot-fullhan`.

Current refs:

- current OpenIPC-native working line: `fh8626v100-mainline@227bcb40f68147864d778f1431973566cca383d8`;
- equivalent native implementation ref: `fh8626v100-openipc-native@227bcb40...`;
- preserved hardware-proven factory-compatible baseline: `fh8626v100-stock-compatible@49fe46e9ddb786e232d1359f9cee68c914a3a8db`.

The native tree is accepted at source/build level. It is **not** yet `HARDWARE_PASS`; the preserved stock-compatible branch remains the recovery/evidence authority until a complete OpenIPC-native cold boot succeeds.

## Current conclusion

The underlying FH8626V100 U-Boot port is hardware-proven on ANJIA AJL33PQ0866 and does not need reimplementation. The agreed architectural transition to an OpenIPC-native installed state is now implemented in the actual U-Boot working line.

The product boundary is explicit:

- factory Fullhan firmware/layout exists only as hardware evidence, migration input and recovery source;
- normal operation uses OpenIPC partition, environment and updater conventions;
- reusable FH8626V100 SoC support is distinct from AJL33PQ0866-specific Boot-ROM/DDR/PHY/persistent-flash data;
- production does not retain factory commands or layout merely for compatibility.

## Hardware-proven recovery baseline

`HARDWARE_PASS` on `fh8626v100-stock-compatible@49fe46e9...` includes:

- modern upstream-U-Boot ARM1176 execution;
- UART console;
- 64 MiB DRAM discovery/relocation;
- timer/autoboot;
- both FH8626 GPIO banks;
- SPI NOR detection and repeated reads at 50 MHz;
- legacy image verification and Linux handoff;
- FH8626 GMAC/MDIO, PHY detection, 100 Mbit/s link, ping and TFTP;
- persistent SPI environment access;
- cold boot through the Fullhan Boot-ROM data container;
- OpenIPC boot through the replacement U-Boot;
- boot of the complete installed factory firmware through the replacement U-Boot.

That path used the original placement:

- Boot-ROM data: `0x00000..0x0ffff`;
- factory environment: `0x10000..0x1ffff`;
- U-Boot: `0x20000..0x4ffff`;
- kernel start: `0x50000`.

This geometry remains documented for recovery only.

## Implemented OpenIPC-native 8 MiB layout

The current `fh8626v100-mainline` uses the standard OpenIPC geometry:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

| Region | Offset | Size |
|---|---:|---:|
| boot | `0x000000` | 256 KiB |
| env | `0x040000` | 64 KiB |
| kernel | `0x050000` | 2048 KiB |
| rootfs | `0x250000` | 5120 KiB |
| rootfs_data | `0x750000` | 704 KiB / remainder |

The standard 256 KiB `boot` partition has one board-specific internal split:

- `0x00000..0x0ffff`: reconstructed Fullhan Boot-ROM/DDR data container;
- `0x10000..0x3ffff`: 192 KiB physical U-Boot slot.

Environment is therefore at the normal OpenIPC offset `0x40000`; kernel still starts at `0x50000`.

## Native Fullhan ROM descriptor

The OpenIPC-native packer no longer preserves the factory descriptor raw size, aligned size or fixed factory JAMCRC as a production requirement.

For each linked `u-boot.bin`, it generates the descriptor from the actual payload:

- flash source offset: `0x10000`;
- raw size: exact linked U-Boot size;
- aligned size: raw size rounded up to `0x100`, limited by the 192 KiB physical slot;
- checksum: JAMCRC calculated over that aligned payload;
- RAM load address: `0xA0800000`;
- entry address: `0xA0800000`.

Alignment padding and the unused tail of the physical U-Boot slot remain erased (`0xff`). The historical fixed factory descriptor values remain in `tools/fh8626_bootchain.py` and stock evidence only.

Observed native build at `227bcb40...`:

- raw U-Boot: `0x2ef20`;
- aligned ROM payload: `0x2f000`;
- JAMCRC: `0x0c6d419b`;
- physical U-Boot slot: `0x30000`;
- aligned-payload headroom inside the physical slot: `0x1000` bytes.

The native artifact inspector accepted:

`uboot flash=0x10000 size=0x2ef20/0x2f000 load=0xa0800000 entry=0xa0800000 jamcrc=0x0c6d419b OK`.

## OpenIPC production environment

The production target uses `CONFIG_ENV_OFFSET=0x40000`, prompt `OpenIPC # `, and current OpenIPC-style NOR variables rather than custom FH8626 update names.

Important contracts:

- `soc=fh8626v100`;
- `board=anjia-ajl33pq0866`;
- `manufacturer=fullhan`;
- `baseaddr=0xa1000000`;
- `flashsize=0x800000`;
- `osmem=39M`, `totalmem=64M`;
- `kernaddr=0x50000`, `kernsize=0x200000`;
- `rootaddr=0x250000`, `rootsize=0x500000`;
- `mtdpartsnor8m`, `setnor8m`;
- `cmdnor`, `bootcmdnor`;
- `updatetool=tftpboot`;
- `ubnor` / `ubwrite`;
- `uknor` / `ukwrite`;
- `urnor` / `urwrite`;
- kernel name `uImage.${soc}`;
- rootfs name `rootfs.squashfs.${soc}`;
- root filesystem `/dev/mtdblock3`;
- `ethaddr=${ethaddr}` propagated into Linux bootargs.

`netboot` uses the same bootargs-expansion path as NOR boot, avoiding unexpanded `${osmem}`, `${mtdparts}` or `${ethaddr}` placeholders.

The board-qualified U-Boot update filename is:

`u-boot-${soc}-${board}-nor.bin`

because the Boot-ROM/DDR data has only been validated on AJL33PQ0866.

The production target does not require factory `kload`, `ethact=FH EMAC` cleanup, factory GPIO command syntax or factory `bootcmd/set_gpio` policy. Those compatibility helpers are behind `CONFIG_FH8626V100_STOCK_COMPAT` and enabled only in the RAM migration/recovery configuration.

## Native artifacts and build contract

`tools/fh8626_openipc_boot.py` emits:

- `u-boot-fh8626v100-anjia-ajl33pq0866-nor.bin` — exact `0x50000` OpenIPC updater image: 256 KiB boot plus an erased 64 KiB environment sector;
- `...-boot.bin` — exact `0x40000` boot partition;
- `...-bootstrap.bin` — 64 KiB reconstructed Boot-ROM data container;
- `...-uboot.bin` — U-Boot padded to the 192 KiB physical slot;
- `...-raw.bin` — linked production U-Boot;
- `...-ram.bin` — non-persistent migration/recovery U-Boot;
- `SHA256SUMS`.

The `0x50000` `-nor.bin` boundary intentionally matches the normal OpenIPC `ubwrite` range up to the kernel offset. Its environment sector must contain only `0xff`, so first boot uses compiled defaults.

`build.sh` is self-validating. It:

1. runs the native artifact unit tests;
2. builds production U-Boot;
3. generates native artifacts;
4. self-inspects the generated `-nor.bin` descriptor/layout/environment;
5. builds the RAM recovery target;
6. creates SHA-256 sums.

Observed build evidence at `227bcb40...`:

- 8 native artifact tests: PASS;
- production U-Boot: PASS;
- native packer/artifact self-inspection: PASS;
- RAM recovery U-Boot: PASS.

The production target retains useful operator/recovery facilities including TFTP upload, TFTP variables, command-line editing, autocomplete, long help, `sleep`, `sf`, `bootm`, memory/CRC, GPIO, MII, ping and TFTP download.

MMC/FAT is intentionally not enabled without a concrete U-Boot recovery requirement. Automatic bootcount/fallback likewise remains future work until an OpenIPC recovery policy/storage contract is defined.

## Existing workflow limitation

The repository GitHub workflow predates the OpenIPC-native artifact scheme. Its final post-build step still looks for old factory-layout filenames and old descriptor values.

The permitted Koba GitHub App does not have `workflows` write permission. Therefore `.github/workflows/build.yml` could not be corrected through the authorized mutation path.

The observed workflow reaches and passes the self-validating `build.sh` stages described above, including both U-Boot builds and native artifact inspection, then reports failure only in the stale legacy post-check.

Do not:

- classify that red final workflow result as a U-Boot compile failure;
- add fake legacy artifacts just to satisfy the old check;
- bypass the Bridge permission boundary.

Update the workflow later using an authorized identity with workflow-write permission.

## Board versus SoC boundary

The native implementation identifies:

- SoC: Fullhan FH8626V100;
- validated board: ANJIA AJL33PQ0866;
- DTS model: `ANJIA AJL33PQ0866`;
- compatibles: `anjia,ajl33pq0866`, `fullhan,fh8626v100`.

Do not publish the current boot artifact as universal FH8626V100. A future second FH8626 board should reuse common SoC drivers/support while supplying and validating its own board data where DDR/Boot-ROM, PHY/reset or flash assumptions differ.

## Kernel-size gate

The remaining architectural gate for the standard OpenIPC 8 MiB map is the final OpenIPC kernel size.

Target: a 2 MiB `uImage` partition.

Required order:

1. reconcile the intended `openipc-linux/fullhan-fh8626v100` source/upstream PR state;
2. build the intended OpenIPC kernel with the relevant final configuration;
3. measure `uImage` exactly;
4. if it fits 2 MiB, keep the standard OpenIPC map;
5. if it does not fit, first review compression/config/built-in/module choices;
6. accept a nonstandard partition map only if the required kernel genuinely cannot fit after reasonable cleanup.

The historical 3 MiB Firmware reservation is not evidence that 3 MiB is required.

## Migration safety

The native layout transition is a one-time destructive conversion. It moves U-Boot from `0x20000` to `0x10000`, discards the factory environment at `0x10000`, creates OpenIPC environment at `0x40000`, and moves rootfs to `0x250000`.

The U-Boot repository documents the staged procedure in `doc/board/fullhan/fh8626v100-openipc-migration.rst`:

1. enter RAM recovery U-Boot;
2. verify SPI/network before writes;
3. write/read-back-compare final kernel;
4. write/read-back-compare final rootfs;
5. erase the new rootfs_data area;
6. load the exact `0x50000` board-qualified `-nor.bin` updater;
7. write/read-back-compare the complete boot + erased-env region last;
8. reset only after all comparisons pass.

The first physical conversion requires a verified full-flash backup, UART and an external SPI programmer. Power loss after destructive writes can require external recovery.

The native state is not `HARDWARE_PASS` until the physical camera cold-boots the complete migrated image and the new MTD/environment/update behavior is exercised.

## Contribution curation

`fh8626v100-mainline@227bcb40...` is the current engineering tree that should be tested on hardware. Its iterative implementation history is not the intended final contribution series.

After hardware acceptance:

- rebuild/squash the native delta into coherent patches while preserving the exact tested tree semantics;
- fold the historical `ethaddr` WIP into its logical board/Linux compatibility patch;
- keep generic DesignWare prerequisites independently reviewable;
- keep Fullhan SPI/Ethernet quirks isolated from board policy;
- run current Das U-Boot checkpatch/style gates;
- preserve `Signed-off-by` provenance in the final series;
- document Boot-ROM reconstruction provenance clearly;
- recheck current OpenIPC source-repository ownership before submission.

OpenIPC currently uses multiple SoC/family U-Boot repositories rather than one universal source tree. FH8626 source stays in `ArthurKoba/u-boot-fullhan` until OpenIPC maintainers choose its organization-level destination. Do not copy U-Boot source into Firmware or Builder.

## OpenIPC integration path

After native hardware acceptance and repository-ownership agreement:

1. publish/consume the board-qualified `0x50000` U-Boot updater artifact according to the naming agreed with OpenIPC;
2. use normal OpenIPC 8 MiB kernel/rootfs offsets if the 2 MiB kernel gate passes;
3. integrate the accepted artifact through Firmware image assembly rather than carrying the historical factory layout;
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
