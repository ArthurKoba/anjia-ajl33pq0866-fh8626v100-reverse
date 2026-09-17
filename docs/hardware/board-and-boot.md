# Board and boot contracts

Evidence classes in this file are explicit; source/reverse findings are not silently promoted to hardware acceptance.

## Confirmed board identity

- SoC: Fullhan FH8626V100.
- Camera/board: ANJIA AJL33PQ0866.
- Sensors: dual GC1054 MIPI.
- Exercised native mode: 1280x720 @ 25 fps.

## Native OpenIPC platform acceptance

`HARDWARE_PASS`: the accepted platform baseline covers native boot, NOR/MTD, reboot, watchdog, GPIO/pinmux, Ethernet, I2C controllers, SADC, bounded PWM, USB host, SD0 microSD, media module/clock ABI and RTX audio on the exercised camera.

Exact scope and AJL33PQ0866 RTC/TSENSOR policy are recorded in `platform-validation.md`.

This is platform-level acceptance and does not imply Majestic or Divinus media/ISP acceptance.

Related subsystem contracts:

- network identity and SD0 wiring/safety: `network-storage.md`;
- U-Boot RAM-chainload/SPI/DMA/kload acceptance: `uboot-port.md`;
- watchdog ownership and recovery semantics: `watchdog.md`;
- illumination, AUTO-light, IR-cut and shutdown behavior: `illumination.md`.

## Stock flash layout

Primary stock flash bytes are external evidence and are indexed through `evidence/MANIFEST.tsv`.

The accepted target partition map is:

| offset | length | role | format |
|---:|---:|---|---|
| `0x000000` | `0x010000` | bootstrap | raw early boot code |
| `0x010000` | `0x010000` | U-Boot environment | raw environment |
| `0x020000` | `0x030000` | U-Boot | bootloader |
| `0x050000` | `0x300000` | kernel | Linux 4.9.129 uImage |
| `0x350000` | `0x080000` | data | JFFS2 |
| `0x3d0000` | `0x080000` | res | JFFS2 |
| `0x450000` | `0x3b0000` | app | SquashFS 4.0 / xz |

Stock `mtdparts` was:

`spi_flash:64k(bootstrap),64k(uboot-env),192k(uboot),3M(kernel),512k(data),512k(res),-(app)`

The kernel payload begins at image offset `0x050040`.

`HARDWARE_PASS`: the reconstructed 8 MiB MX25L6405D recovery image restored cold boot on the physical camera. That proves the recorded recovery image/layout for the exercised board; it does not make raw partition derivatives independent authorities.

Historical U-Boot IP values are evidence from stock configuration only and do not define current OpenIPC network policy.

## Cold-boot dual-sensor prerequisite

`HARDWARE_PASS`: TELE visibility depends on the validated GPIO5 bootstrap sequence: GPIO5 LOW before the required Fullhan media-module initialization sequence, then GPIO5 HIGH before sensor/media startup.

This prerequisite is separate from runtime logical lens switching on GPIO4/GPIO14.

## PTZ

`HARDWARE_PASS`: the exercised motor backend uses `/dev/fh_pwm`. Stock-style type-2 pan/tilt movement was physically accepted after correcting swapped motor connectors.

Higher-level HTTP/ONVIF integration was also exercised. Do not reopen the electrical PWM reverse unless contradictory target evidence appears.

## Repository boundary

Kernel/platform implementation belongs in the relevant OpenIPC repository. This camera repository records the target contract, validation state and evidence boundary.

Low-level reverse questions are handled in the canonical Ghidra MCP project when a concrete implementation blocker requires them.
