# FH8626V100 production kernel-config audit

Status: `SOURCE_CONFIRMED / BUILD_GATE`.

Checked: 2026-09-18.

This audit covers the OpenIPC Firmware kernel configuration for FH8626V100 after the Linux contribution series was separated from Firmware. It is deliberately based on the actual FH8626/AJL contracts and selected runtime, not on copying another Fullhan kernel config.

## Current refs

- Linux source: `ArthurKoba/openipc-linux/rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`.
- Firmware candidate before this pass: `rework/fh8626v100-clean-integration@f9146dd42a2f606d305ebccd301268848de26880`.
- Firmware candidate after this pass: `rework/fh8626v100-clean-integration@c437d6eb62ade81595c20cbb765b8ad10300e3e7`.
- Working/audit branch: `rework/fh8626v100-production-kconfig@c437d6eb62ade81595c20cbb765b8ad10300e3e7`.
- ANJIA Builder profile: `ArthurKoba/openipc-builder/rework/fh8626v100-anjia-clean-profile@1ea41ef2dc9a38e138907eda6d316bb743631ebe`.
- Divinus reference: `ArthurKoba/openipc-divinus/fh8626v100-canonical@1e624bd5aca97ba772413d2b00a10314d1db039f`.

Only `br-ext-chip-fullhan/board/fh8626v100/fh8626v100.generic.config` changed in the configuration pass. No Linux, Builder or Divinus implementation was copied into Firmware.

## Method

Another Fullhan config is not an authority. Mature HiSilicon/Goke/Ingenic OpenIPC ports were checked only for repository/layout conventions. Their individual kernel configs disagree on several debug, configfs, keyring and recovery choices, so their symbol values are not treated as defaults.

A symbol is changed only when its role can be tied to the current target contract:

- `PRODUCTION_REQUIRED`: needed for boot, storage/rootfs, network, hardware or selected runtime ABI;
- `BOARD_OPTIONAL`: a board can enable the function through its Builder fragment;
- `BRINGUP_ONLY`: intentionally retained for recovery/development;
- `UNRESOLVED`: do not change until the consumer is known;
- `REMOVE`: no selected target consumer or platform instance remains.

The goal is a correct, maintainable config, not minimum line count.

## Applied cleanup

### REMOVE

The following options were removed or made explicitly disabled:

| Symbol | Reason |
| --- | --- |
| `CONFIG_USELIB` | Obsolete libc5-era `uselib(2)`; Buildroot uses musl and normal mmap-based dynamic loading. |
| `CONFIG_IRQ_DOMAIN_DEBUG` | Exposes IRQ-domain mapping only through debugfs; no runtime function depends on it. |
| `CONFIG_FTRACE` | Kernel tracing infrastructure; no selected production consumer. |
| `CONFIG_CONFIGFS_FS` | No selected FH8626/AJL userspace or kernel feature creates configfs items. |
| `CONFIG_HOSTAP` (+ firmware options) | Legacy Intersil Prism2/2.5/3 stack, unrelated to the board RTL8188FU USB adapter. |
| `CONFIG_USB_WUSB_CBAF` | Wireless-USB cable-association support; unrelated to the target USB host role. |
| `CONFIG_USB_WDM` | CDC Wireless Device Management for phones/modems; no target device. |
| `CONFIG_USB_ANNOUNCE_NEW_DEVICES` | Extra enumeration logging only. |
| `CONFIG_USB_DYNAMIC_MINORS` | Not required by the target's fixed small USB device set. |
| `CONFIG_FH_CLK_MISC` | Vendor userspace clock-control miscdev. Kernel drivers use the clock framework; current ANJIA board support and FH8626 Divinus path do not consume this ABI. |
| `CONFIG_DNS_RESOLVER` | Kernel DNS upcall facility for CIFS/AFS; neither filesystem is selected. |
| `CONFIG_KEYS` | No remaining selected keyring consumer after DNS resolver removal and explicit NFSv4 disablement. |
| `CONFIG_CRYPTO_USER` | No selected userspace kernel-crypto configuration consumer. |
| `CONFIG_CRYPTO_USER_API_SKCIPHER` | No selected AF_ALG symmetric-cipher consumer. |
| `CONFIG_CRYPTO_SEQIV`, `CONFIG_CRYPTO_CBC`, `CONFIG_CRYPTO_DES` | Removed as manually forced legacy crypto pieces; real kernel consumers may select needed primitives automatically. |
| `CONFIG_CRYPTO_DEV_FH_AES` | The current FH8626 non-DT machine registers no FH AES platform device and the open runtime does not consume the driver. |

The generic RTC **device** selection was also changed from on to opt-in:

`CONFIG_FH8626V100_ONCHIP_RTC=n`

This is not removal of the shared RTC driver. `CONFIG_RTC_CLASS` and `CONFIG_RTC_DRV_FH` remain available so a board profile can enable the on-chip device after its RTC path is validated.

### Recovery networking made explicit

NFS is intentionally retained as a bring-up/recovery facility, but the config now states the actual intended protocol scope:

- `CONFIG_NFS_FS=y`
- `CONFIG_NFS_V2=n`
- `CONFIG_NFS_V3=y`
- `CONFIG_NFS_V4=n`

NFSv4 is not part of the target recovery contract and would add GSS/keyring machinery. `CONFIG_BLK_DEV_INITRD`, `CONFIG_IP_PNP` and DHCP remain for the current CPIO/TFTP/recovery workflow. These are not evidence that the normal NOR rootfs depends on NFS or initramfs.

## Retained production functions

These are deliberately not removed merely to reduce size:

- `MTD/M25P80/SPI_NOR`, JFFS2, OverlayFS, SquashFS/XZ: the standard 8 MiB NOR/rootfs contract.
- `FH_GMAC/FH_GMAC_DA/FIXED_PHY`: accepted RMII Ethernet path.
- `USB/DWC2/VBVALIDOVEN`: target USB host path used by the soldered Wi-Fi adapter.
- `CFG80211/WEXT/RFKILL`: current OpenIPC RTL8188FU driver builds its cfg80211 interface and the ANJIA profile selects that driver.
- `MMC/VFAT/NLS`: current ANJIA product has the proven one-bit microSD path. The actual controller/wiring selection remains in the Builder fragment through `MMC_FH` and `FH8626V100_SD0_1BIT`.
- `FH_SADC_V2/FH_SADC_V21`: one driver with v2.1 feature blocks, not duplicate drivers; SADC is consumed by the illumination/light-sensor path.
- `FH_PINCTRL_MISC_DEV`: production board support writes `/proc/driver/pinctrl` for PTZ/illumination mux policy.
- `PWM/PWM_FULLHAN`: production PTZ backend.
- `FH_AXI_DMAC`: part of the curated platform implementation; still requires owner build/hardware retest before final acceptance.
- `I2C/I2C_CHARDEV/I2C_FH_INTERRUPT`: retained for sensor/control integration while the source-owned sensor path is being completed.
- `FH_EFUSE`: retained until the final identity/calibration/secure-data consumers are fully closed.
- `WATCHDOG/FH_WATCHDOG`: accepted platform function.
- `MODULES/MODULE_UNLOAD`: required by external/device-selected modules such as the RTL8188FU driver.

## Intentionally not minimized yet

The following stay enabled pending product/build evidence:

- `CONFIG_SYSVIPC` and `CONFIG_POSIX_MQUEUE`: no current FH8626 source dependency was found, but streamer/product compatibility is not closed and their removal is not needed before the build gate.
- `CONFIG_DEBUG_FS`: retained during platform/media bring-up. Revisit after the complete source-owned runtime is hardware-accepted.
- `CONFIG_PRINTK_TIME`: retained because boot/driver timing remains useful during the hardware acceptance phase.
- `CONFIG_I2C_CHARDEV`: keep until the final GC1054/MIPI source path and its transport boundary are settled.
- NFS/IP autoconfiguration/initrd: remain recovery features until the owner decides that a separate bring-up kernel fragment is worth maintaining.

This list prevents later agents from treating retained options as accidental just because a different platform omits them.

## Validation gate

No authoritative kernel/Buildroot build was run in this pass.

The next owner build must use exact Firmware `rework/fh8626v100-clean-integration@c437d6eb62ade81595c20cbb765b8ad10300e3e7` and Linux `357c2d13...`, then record:

1. final resolved kernel `.config`;
2. `uImage` size and SHA-256;
3. SquashFS size and SHA-256;
4. confirmation that `uImage <= 2048 KiB` and SquashFS `<= 5120 KiB`;
5. boot/MTD/rootfs_data;
6. Ethernet;
7. USB + RTL8188FU association;
8. microSD/VFAT;
9. SADC illumination input;
10. PWM/PTZ and pinctrl runtime;
11. watchdog;
12. media/audio startup using the actually selected source/runtime path.

After build resolution, inspect the generated `.config` for any symbol that Kconfig re-selected despite the requested `n`. Such automatic selections are dependencies to document, not lines to fight blindly.
