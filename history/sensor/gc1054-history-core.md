# Historical GC1054 and lens findings

This file preserves useful historical sensor observations. Current authority is `../../docs/sensor/gc1054-and-lens-switch.md` and `../../docs/sensor/vendor-mipi-abi.md`.

## Hardware boundary

Camera uses dual GC1054 MIPI sensors. Exercised native mode is 1280x720@25. Product default lens is WIDE; stock exposes WIDE/TELE logical lenses.

GPIO5 cold-boot sequencing is a board prerequisite for TELE visibility and is distinct from runtime logical lens switching.

## Stock sensor library boundary

Captured stock runtime mapped `libgc1054_mipi.so` and `libmipi.so`. Exact target analysis showed sensor access through the vendor sensor/I2C wrapper with ioctl values `0x704`, `0x706` and `0x707`; `Sensor_Create` publishes the stock callback table.

One captured `libsensor.so` payload was byte-identical to `libgc1054_mipi.so`, so duplicate copies must not be treated as independent evidence.

## Lens-switch lessons

One source generation incorrectly treated direct GC1054 callback invocation as equivalent to the stock `API_ISP_SensorInit()` wrapper. Reverse evidence showed the wrapper also obtains/masks chip ID and propagates it before sensor initialization.

Historical WIDE -> TELE -> WIDE testing also showed that switching could leave transient exposure values when no automatic AE provider restored lens state. A later workaround cached/restored per-lens integration and gain. This remains historical behavior, not current product acceptance.

## Historical MIPI MMIO observation

A 2026-08-25 OpenIPC checkpoint captured point-in-time MMIO values for the `0xF0000000`, `0xF0002000`, `0xF1000000` and `0xF1100000` MIPI-facing windows.

These observations corroborated regions independently identified by target reverse, but they are not portable initialization tables and must not be replayed blindly.

## Selector ordering

Stock UART evidence confirmed the captured selector order:

- WIDE: assert GPIO4 before deasserting GPIO14;
- TELE: assert GPIO14 before deasserting GPIO4.

Confirmed endpoint mapping remains WIDE GPIO4=1/GPIO14=0 and TELE GPIO4=0/GPIO14=1.

## TELE day-to-night observation

A 2026-08-28 TELE day->night transition showed substantial I2C0, ISP and PAE activity while PWM remained unchanged. This supports sensor/control and ISP/media activity without motor-PWM activity in the sampled transition, but does not identify the exact ioctl/register sequence.

## Stock interface ownership

A stock interface probe captured Apollo with descriptors open to `/dev/i2c-0`, `/dev/pae`, `/dev/jpeg`, `/dev/isp`, `/dev/bgm`, `/dev/vmm_userdev`, `/dev/rtxbus`, `/dev/fh_sadc`, `/dev/fh_pwm`, `/dev/media_process` and `/dev/watchdog`.

This is runtime ownership/context evidence only. Descriptor presence does not identify the operation implementing a feature.

## Historical diagnostic leads

Read-only GC1054 ID probes (`F0=0x10`, `F1=0x54`) were used historically to distinguish physical sensor reachability from later sensor-format/media failure.

A separate focused reverse lead identified `/dev/isp` ioctl `0x40046931` before sensor callback initialization. Its argument semantics and causal relation to TELE recovery were not established, so it remains a lead rather than hardware acceptance.

## Boundary

No broad sensor reverse is currently requested. Reopen only for a concrete Majestic/native integration blocker or contradictory target evidence, using the canonical Ghidra MCP project.
