# FH8626V100 engineering chronology

This is the durable chronological history of the camera work. It records what was attempted, what was proven, what failed and which result changed direction. It does not describe obsolete repository/workspace structure.

## Reading rule

- `CONFIRMED` = backed by target/hardware evidence or exact target reverse/source evidence.
- `HOST PASS` = proved only on host/offline tests.
- `TARGET PASS` = executed successfully on target, but not automatically a physical subsystem acceptance.
- `INVALID/FAILED` = did not establish the intended claim.
- Later evidence supersedes earlier interpretation without erasing the earlier step.

## 2026-08-27 — stock boot and media topology

- `CONFIRMED`: FH8626V100, Linux 4.9.129, Fullhan ARM uClibc toolchain.
- Stock U-Boot loads the kernel from SPI offset `0x00050000` to `0xA1000000`; the uImage load/entry address is `0xA0008000`.
- Effective SPI NOR layout: 64 KiB bootstrap, 64 KiB environment, 192 KiB U-Boot, 3 MiB kernel, 512 KiB data, 512 KiB res, remaining app.
- Stock exposes 14 PWM channels, three I2C controllers and Fullhan GMAC.
- Stock media startup confirms dual GC1054 MIPI operation and logical WIDE/TELE lens selection.
- Native sensor mode is 1280x720 while stock publishes a 1920x1080 main stream and 640x360 secondary stream, proving downstream scaling.

## 2026-08-28 — owner/runtime candidate and lens-switch boundary

A large native owner/runtime candidate was assembled around dual GC1054, day/night/white-light profiles, AE/AWB/CCM and runtime ISP blocks. It was an implementation candidate, not an acceptance result.

Hardware lens-switch testing exposed a concrete state problem: WIDE -> TELE -> WIDE could leave transient exposure values when no automatic AE provider restored lens state. A later workaround cached per-lens integration/gain. TELE sensor-write failures remained a separate issue.

A second correction restored the stock-facing `API_ISP_SensorInit()` wrapper instead of directly invoking the GC1054 init callback. Reverse evidence showed the wrapper also acquires/masks chip ID and propagates state before sensor initialization.

## 2026-08-30 — Divinus external encoded-source path

The first Divinus integration deliberately used an external encoded H.264 source above hardware-HAL ownership instead of pretending that an FH8626 native HAL was already known.

FH86 framing, bounded stream assembly, session-scoped generation handling, AF_UNIX transport and reconnect semantics were implemented before application integration.

`HOST PASS`: focused wire/stream/source tests and host/ARM builds passed. A synthetic FH86 -> Divinus -> RTSP/TCP -> RTP/H.264 path was proven end-to-end on host. Physical FH8626 acceptance was not claimed.

## 2026-08-30 — native-HAL work remained bounded

Native FH8626 work started only after the external-source architecture was separated cleanly.

Exact rule: do not import vendor `.so` files as an upstream implementation and do not copy FH8852-family structure layouts as FH8626 ABI.

Offline tests exercised encoded-descriptor ownership, ring wrap, release semantics and retryable teardown. Production acquisition remained blocked until exact target device/ring/pipeline/force-IDR/rate-control contracts were known.

## 2026-08-29 — exact target sensor/MIPI boundary

`CONFIRMED`: captured stock runtime mapped `libgc1054_mipi.so` and `libmipi.so`.

Target analysis identified `libmipi.so` MMIO windows around `0xF0000000`, `0xF0002000`, `0xF1000000` and `0xF1100000`, and GC1054 sensor access through the stock sensor/I2C wrapper using ioctl values `0x704`, `0x706` and `0x707`.

One captured `libsensor.so` payload was byte-identical to `libgc1054_mipi.so`; duplicate copies were therefore not independent evidence.

## 2026-08-27/29 — stock flash and watchdog

`CONFIRMED`: stock bootstrap, U-Boot and kernel captures established the effective boot environment and partition layout. Live bootstrap/environment state outranks fallback strings compiled into U-Boot when they differ.

Stock watchdog observations showed Apollo owning `/dev/watchdog`; stop/start actions closed and reopened that descriptor without restarting Apollo. No `/sys/class/watchdog` control surface was present on the captured stock build.

## 2026-08-29 — invalid targeted kernel slices rejected

Several address-targeted watchdog/restart disassembly slices were reviewed and rejected because they decoded data as code, produced incoherent control flow and lacked reliable anchors.

Exact target symbol maps and runtime evidence remained stronger authority. This became a durable rule: persuasive filenames or guessed addresses are not reverse evidence.

## 2026-09-04/05 — platform and media integration matured

The platform side reached hardware-accepted native OpenIPC boot, Ethernet, storage, watchdog, GPIO/pinmux, PWM, USB, media-module clocking and RTX audio boundaries documented in current hardware pages.

Media/ISP integration continued to improve, but a 1080p surface/SPS/decode success remained weaker than complete image/DDR/cadence/RC acceptance. Streamer and ISP acceptance therefore stayed separated from platform acceptance.

## Current historical boundary

Broad stock reverse is no longer a standing activity. Existing chronology is retained only to explain current contracts and avoid repeating failed directions. New reverse work begins from a concrete implementation or validation blocker and is performed in the canonical Ghidra MCP project.
