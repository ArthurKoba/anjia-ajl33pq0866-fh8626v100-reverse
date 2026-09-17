# FH8626V100 platform validation

This file records the strongest retained camera-level platform acceptance. It is hardware evidence for the exercised baseline, not a claim that the historical validation image is the current product firmware.

## Accepted native OpenIPC platform baseline

`HARDWARE_PASS` on ANJIA AJL33PQ0866 covered:

- native FH8626V100 boot;
- SPI NOR/MTD reads and exact partition registration;
- reboot through the validated PMU reset path;
- watchdog expiry plus boot-time feeder recovery;
- GPIO/pinmux operation, including the shared GPIO23/SADC1 white-light mux sequence;
- Ethernet link and real camera traffic;
- I2C0/I2C1/I2C2 controller registration;
- functional SADC light input;
- bounded PWM output;
- USB host;
- the AJL33PQ0866 SD0 microSD slot;
- media clock/module ABI needed by the camera stack;
- the board RTX audio path.

I2C acceptance at this stage was controller-level only unless a separate device-level test is cited.

## Historical integration/build gate

The accepted clean-apply/build run reported:

- firmware kernel patches matched the tested Linux working tree byte-for-byte;
- the series applied to the tested Linux base without rejects, offsets or fuzz;
- sanitized kernel rebuild and complete `make BOARD=fh8626v100_lite all` packaging passed;
- produced `uImage`: 1,583,456 bytes, SHA-256 `eb7acb04c3b7524450d8b9652b38aab9cf3872c00b2172c5ad3fa1f9023d40c5`;
- produced `rootfs.squashfs`: 2,834,432 bytes, SHA-256 `0d27edc6132d06853edb567997f766baa62926dc239d4a8061db159465a2981a`;
- Linux release was `4.9.129`.

These hashes identify that acceptance build only. Future integration must use the relevant component repository as source authority and be rebuilt/retested from the verified current base. Known preservation refs are recorded in `docs/process/upstream-integration.md`.

## AJL33PQ0866 RTC / TSENSOR policy

The board deliberately disables RTC and TSENSOR in production.

Native and untouched stock Linux 4.9.129 both timed out through the same RTC command-core handshake. Stock product configuration also selected `hw_rtc=no`. Therefore the failure is not evidence of an OpenIPC-only regression.

Generic FH8626V100 RTC support remains valid for other boards with suitable clock/backup hardware. For AJL33PQ0866:

- keep `CONFIG_FH8626V100_ONCHIP_RTC=n`;
- do not expose a fake `rtc0`;
- do not enable TSENSOR merely because vendor conversion code exists;
- do not publish thermal/hwmon temperature without changing, plausible raw samples and board-level proof;
- do not perform speculative PMU/RTC writes to force the block alive.

This is a negative board policy backed by stock/native parity, not a claim that generic FH8626V100 RTC/TSENSOR IP is universally broken.

## Scope boundary

The platform baseline is independent of Majestic vs Divinus. Streamer/media acceptance remains governed by its own evidence ladder.
