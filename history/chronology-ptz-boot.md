# PTZ, boot and platform chronology

This file preserves durable historical observations for PTZ, boot and low-level platform behavior. Current authority is under `docs/hardware/` and `docs/ptz/`.

## 2026-08-29 — PTZ/PWM motor boundary

`CONFIRMED`: exact target ioctl capture showed stock PTZ motor control using request `0xC004700C` on the PWM device path. The observed LEFT payload began with PWM channel `11`; UP began with channel `5`. These directions align with the later hardware-accepted motor map.

IRQ snapshots around movement showed PWM interrupt activity increasing while I2C counters stayed unchanged, supporting a PWM-backed motor path rather than sensor-I2C ownership.

The capture was narrowly filtered, so it is exact positive evidence for the observed events, not proof that no other control path exists.

## 2026-08-28 — stock platform surface

Stock `dmesg` confirmed Linux 4.9.129, effective `mem=39M` and the seven-partition SPI layout. The Fullhan PWM driver exposed 14 channels; GMAC and three I2C controllers were registered and active.

The stock `/dev` surface included `/dev/fh_pwm`, `/dev/isp`, `/dev/jpeg`, `/dev/pae`, `/dev/media_process`, `/dev/watchdog`, `/dev/vmm_userdev` and `/dev/i2c-0..2`.

Stock process state included `dev_ctrl`, `apollo`, `noodles`, `wpa_supplicant` and media workers such as `jpeg_kick`, `vpu_task`, `H264_CB` and `pae_proc`.

A stock `kallsyms` snapshot provided coherent non-zero boundaries for PMU watchdog/restart and PWM functions. These exact target symbols were more reliable than guessed address slices.

## 2026-08-25 — OpenIPC RAM boot on stock kernel

`CONFIRMED`: early OpenIPC bring-up retained the stock FH8626 Linux 4.9.129 kernel. U-Boot loaded the stock kernel at `0xA1000000` and an external OpenIPC initramfs at `0xA1400000`, adding `rdinit=/sbin/init`.

The resulting boot brought up the FH8626 platform, including PWM, I2C, SPI NOR partitions and Ethernet. This milestone proved Linux/OpenIPC RAM bring-up only; it did not prove the later sensor/media/encoder pipeline.

## 2026-08-28 — TELE day/night observation

A TELE day->night capture showed substantial I2C0, ISP and PAE counter growth while PWM remained unchanged. This is observation-only evidence that the sampled transition exercised sensor/control and ISP/media activity without motor PWM activity.

It does not identify the responsible ioctl/register sequence.

## Platform lesson

Target filesystem/control surfaces are vendor-embedded and should not be assumed to match a conventional distro. Actual device nodes, process ownership, target symbols and observed runtime behavior outrank desktop-Linux expectations.
