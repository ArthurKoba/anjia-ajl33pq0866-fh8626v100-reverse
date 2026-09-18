# FH8626V100 kernel contribution-series audit

Status: `ACTIVE / SOURCE_AUDIT`.

Checked: `2026-09-18`.

This file is the camera-project authority for the current FH8626V100 Linux history cleanup. It records cross-repository state and review decisions. Do not copy it into `OpenIPC/linux`, Firmware or Builder.

## Branch roles

Repository: `ArthurKoba/openipc-linux`.

- verified series base: `fullhan-fh8852v200@ee1ef844294bfa1ff15b2f0522d35c987a16a220`;
- PR-facing/integration branch: `fullhan-fh8626v100@0dfafa643770d78389e444c03f46f1711662eda6`;
- isolated OpenIPC MTD correction: `fix/fh8626v100-openipc-mtd-layout@28a923a9d9598a9a4e2c6c0ee4b2eee26698731e`;
- exploratory reconstruction workspace: `rework/fh8626v100-clean-series@868bdd8ddde7a35c2c744e5706941d5e1f9faadf`;
- curated staging series: `rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`.

The PR-facing branch remains untouched during this audit. The exploratory `clean-series` contains audit-time corrections and is not submission history. The separate `final-series` has already been rebuilt from the verified base with no fixup/audit commits and is the source candidate for owner validation.

## Migration chronology

- 2026-09-04: FH8626V100 entered the Linux fork as two broad commits, `8a408dd5` and `0dfafa64`.
- 2026-09-10: Builder began the named-device migration.
- 2026-09-13: the pre-Majestic Builder checkpoint was stabilized.
- 2026-09-16: Firmware preserved the remaining mixed FH8626 state in one WIP snapshot.

The Firmware snapshot retained three kernel patch files (platform, GMAC MAC, DWC2), but the platform patch still mixed SoC enablement with generic clock/pinctrl/SADC/PWM/RTC/PHY work. Therefore the two Linux commits and the three Firmware patches are preservation/history references, not an acceptable final patch structure.

## Evidence boundary

The existing platform baseline remains `HARDWARE_PASS` for the tested ANJIA AJL33PQ0866 tree. Retained hardware evidence covers native boot, SPI NOR/MTD, PMU reboot, watchdog, GPIO/pinmux, Ethernet, I2C controller registration, SADC light input, PWM, USB host, one-bit SD0, media clock/module ABI and RTX audio.

The accepted historical build produced:

- `uImage`: 1,583,456 bytes;
- SHA-256: `eb7acb04c3b7524450d8b9652b38aab9cf3872c00b2172c5ad3fa1f9023d40c5`;
- Linux release: 4.9.129.

That proves the historical tested tree fit a 2 MiB kernel partition. It does **not** promote the reconstructed series to build or hardware acceptance. The reconstructed tree remains `SOURCE_CONFIRMED / AUDIT` until the owner performs the authoritative build and, for behavior changes that require it, the applicable hardware retest.

## Findings and decisions

### OpenIPC MTD layout

The original Linux machine description retained factory-derived Linux partitions (64 KiB bootstrap, 64 KiB env, 192 KiB U-Boot, 3 MiB kernel, 1 MiB rootfs_data, remaining rootfs). That conflicts with the native OpenIPC U-Boot contract and changes Linux MTD numbering.

The final series must introduce the OpenIPC layout directly rather than first adding the obsolete layout and repairing it later:

`256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`

This keeps rootfs at `/dev/mtdblock3` and leaves 704 KiB for rootfs_data.

### Required Fullhan boardconfig input

The Fullhan base contains a top-level `boardconfig` target which copies:

`arch/arm/mach-fh/$(PROJECT_NAME)/board_config.$(PROJECT_NAME).appboard`

to:

`arch/arm/mach-fh/include/mach/board_config.h`

before `vmlinux` is built.

Therefore `board_config.fh8626v100.appboard` is a required build input, not dead vendor debris. An audit-time attempt to remove it was detected as incorrect and reversed before finalization.

The reconstructed form follows the existing Fullhan non-DT model: `iopad.h` contains the recovered FH8626 SoC pin database, while `CONFIG_PINCTRL_SELECT` in the boardconfig chooses the initial board mux profile. The selected 25 device names were mechanically checked against the recovered `PINCTRL_DEVICE` table; all resolve.

The currently known initial profile is the hardware-tested AJL33PQ0866 profile. Do not describe it as universal FH8626 board wiring until another board validates it.

### SD0 policy

The original kernel option `CONFIG_FH8626V100_AJL33PQ0866_MMC` embeds a retail camera name in the SoC kernel Kconfig.

The reconstruction uses the hardware-oriented `CONFIG_FH8626V100_SD0_1BIT` option. It selects the proven one-bit SD0 platform data and `SD0_1BIT_NO_WP` mux. The named device profile remains responsible for enabling it.

The clean Firmware candidate no longer carries an AJL fragment at all. The named ANJIA Builder staging profile `rework/fh8626v100-anjia-clean-profile@1ea41ef2...` owns this selection and uses `CONFIG_FH8626V100_SD0_1BIT=y`. The preserved Firmware/Builder snapshots keep the old symbol only as history.

### Clock phase handling

The shared Fullhan phase setter ORed a new phase into the old field without clearing stale bits, the getter had an operator-precedence bug, and the setter returned 1 on successful clock-API completion.

Keep the generic fix as its own commit because FH8626 SD drive/sample phase programming depends on deterministic field replacement.

### Pinctrl register-state lifetime

The shared Fullhan pinctrl code stored `PinCtrl_Pin.reg` as a pointer to a function-local register variable and retained the pin object for later operations. The reconstruction gives the pinctrl object persistent per-pad register storage.

Keep this as an independent generic correctness fix.

### JL1101 and RMII

The target reports JL1101 PHY ID `0x937c4024`. The reconstructed driver accepts `0x937c4023` through `0x937c4027` and uses the existing RTL8201-style RMII programming path. Older OpenIPC Fullhan branches already carry the `0x937c4023` precedent.

Preserve the FH8626-specific static RMII path from the hardware-tested tree. Early non-DT pinctrl already installs the complete RMII mux and external PHY reset pin; dynamically backing up/switching the mux during PHY probing is not part of the accepted baseline.

The stale bring-up comment claiming the FH8626 iopad table was still pending is removed.

### GMAC MAC propagation

The board's accepted network contract passes standard U-Boot `ethaddr` to the kernel and then into GMAC platform data.

Keep early `ethaddr` parsing, rejection of invalid/sentinel addresses, platform-data fallback when `eth_platform_get_mac_address()` supplies no address, copying the selected address into `net_device`, and validation of SIOCSIFHWADDR before changing private driver/hardware state. The required helper exists in this 4.9 tree.

### GMAC checksum features

Linux 4.9 explicitly documents that `NETIF_F_HW_CSUM` must not be advertised together with `NETIF_F_IP_CSUM` or `NETIF_F_IPV6_CSUM`. Retain removal of the redundant IPv4-specific flag as an independent network correctness commit.

### PWM backend and robustness

The Fullhan base Kconfig already contains a dormant `MACH_FH8626V100` block that defines `FH_PWM_NUM` and selects `PWM_FULLHAN_V21`. Once the FH8626 machine symbol exists, that Kconfig path is sufficient.

The actual missing link was in `drivers/pwm/Makefile`: `CONFIG_PWM_FULLHAN_V21` did not build `pwmv2.o`, which supplies the common driver's low-level helpers. The curated series adds only that Makefile wiring.

During the exploratory audit, FH8626 was also added to the preceding `ARCH_FH885xV200 || ARCH_FH865x` Kconfig block; that duplicated the `FH_PWM_NUM` definition. The mistake was detected and is absent from `final-series`.

Keep the separately reviewable shared PWM hardening from the tested tree.

### AXI DMA registration

The current Firmware kernel config selects `CONFIG_FH_AXI_DMAC=y`.

In non-DT mode, `fh_axi_dma_adapt.c` binds to a platform device named `fh_axi_dmac` and requires MMIO, IRQ and `fh_axi_dma_platform_data`. The original FH8626 board file registered only the legacy `fh_dmac` device under `CONFIG_FH_DMAC`, so an AXI-DMAC build had no device to probe.

The reconstruction now registers `fh_axi_dmac` under `CONFIG_FH_AXI_DMAC` with the recovered `DMAC_REG_BASE`, `DMAC0_IRQ`, ascending priority and `ahb_clk`, matching the existing Fullhan non-DT pattern. Legacy `FH_DMAC` support remains separate.

### RTC / TSENSOR

AJL33PQ0866 deliberately keeps RTC/TSENSOR disabled. Native and untouched stock 4.9 both timed out on the same RTC command path.

Keep generic FH8626 RTC registration available but opt-in (`CONFIG_FH8626V100_ONCHIP_RTC=n` by default). Keep the shared RTC error-propagation fix separate so command failures cannot be decoded as unsigned timestamps.

### DWC2 VBUS policy

Fullhan platform data uses `0xffffffff` as the no-controllable-VBUS-GPIO sentinel. The function already treats that value as a no-op. Retain the warning-to-debug change as a small independent DWC2/Fullhan patch.

### SADC cleanup rejected

Do not carry the migration-only `strncpy(..., "sadc", sizeof("sadc"))` to `strlcpy` change. The literal copy already includes its terminating NUL and the edit is unrelated to FH8626 enablement.

The FH8626 SADC v2/v2.1 Kconfig paths themselves are correctly gated on `MACH_FH8626V100`.

### Generated/legacy style

Do not mass-reformat recovered `iopad.h` or unrelated legacy Fullhan code merely to make checkpatch quieter. Existing generated pinctrl declarations are intentionally dense, and pre-existing legacy whitespace/FIXME lines are not part of the FH8626 contribution unless a touched semantic change requires them.

## Curated final staging series

`rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e` contains 13 commits over `fullhan-fh8852v200`:

1. `a68a09e` — `clk: fullhan: fix phase field handling`
2. `0a0dae2` — `pinctrl: fullhan: keep register state in persistent storage`
3. `ae9f464` — `pwm: fullhan: build the v2 backend when selected`
4. `eb6f3eb` — `ARM: fullhan: add FH8626V100 SoC data`
5. `2b7a6c6` — `ARM: fullhan: add FH8626V100 machine description`
6. `a1d8bdd` — `ARM: fullhan: enable the FH8626V100 platform`
7. `f2c06c6` — `ARM: fullhan: register the FH8626 AXI DMA controller`
8. `35ed593` — `net: fullhan: support JL1101 PHY variants on FH8626V100`
9. `dbc0038` — `net: fullhan: accept a MAC address from platform data`
10. `0b8f45a` — `net: fullhan: avoid contradictory checksum features`
11. `4d04535` — `pwm: fullhan: harden configuration and status handling`
12. `c8de741` — `rtc: fullhan: propagate command failures to the RTC core`
13. `357c2d1` — `usb: dwc2: treat the Fullhan no-VBUS-GPIO sentinel as optional`

The final-series tree was compared file-by-file against the exploratory clean-series. Every functional blob is identical. The only two differing blobs are `board.c` and `board_config.fh8626v100.appboard`, where final-series removes ANJIA-specific wording from comments without changing code.

No SADC cleanup, fork-local metadata or audit documentation is present in Linux.

## Remaining gates

1. Owner-side contribution validation of `rework/fh8626v100-final-series`: repository/kernel checks, exact OpenIPC build and final `uImage` measurement.
2. Update the Firmware AJL kernel fragment from `CONFIG_FH8626V100_AJL33PQ0866_MMC` to `CONFIG_FH8626V100_SD0_1BIT` only when Firmware is switched away from its preserved old kernel patch set to the curated Linux series.
3. Remove Firmware-side duplicate kernel patches from the eventual upstream Firmware contribution once the Linux branch is authoritative.
4. Retest behavior-changing source deltas as applicable, with special attention to AXI-DMA registration, before promoting the reconstruction beyond `SOURCE_CONFIRMED / AUDIT`.
5. Keep the 2 MiB kernel partition target; audit config/compression/built-ins before any layout change if the final image does not fit.
6. Only after those gates, replace the PR-facing branch once if the owner still wants the existing upstream submission history rewritten.
7. Agents remain browser/API-first and do not create a duplicate local build environment unless explicitly asked.

The exact upstream OpenIPC FH8626 pull-request number/status remains unverified through the currently accessible repository API. Do not invent it or treat local branch existence as proof of upstream acceptance.
