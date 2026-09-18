# FH8626V100 Divinus cleanup handoff

Checked: `2026-09-18`.

## Result

The active Divinus development line is:

- repository: `ArthurKoba/openipc-divinus`
- branch: `work/fh8626v100`
- candidate: `168b2ecfeffcb53c2ed2a1d86c4897fdd3423820`
- evidence status: `SOURCE_CLEAN_CANDIDATE / HARDWARE_PENDING`

The matching Firmware test direction is:

- repository: `ArthurKoba/openipc-firmware`
- branch: `work/fh8626v100-divinus`
- candidate: `d589e0546384a4fc094cc959f81d6f2864ce997f`
- role: temporarily pin the exact Divinus candidate for attributable hardware testing.

## What was removed as legacy

The current Divinus line no longer contains the external FH86 encoded-source architecture:

- `source: fh86` configuration mode;
- Unix socket source client;
- FH86 wire framing/protocol;
- separate H.264 source adapter;
- external-owner control FIFO path;
- the associated legacy transport/source/wire tests.

The weak `fh8626_native_board_prepare()` hook was also removed. Generic Divinus no longer offers an insertion point for AJL-only reset/lens/GPIO policy.

These removals do not erase historical evidence. The old transport baseline remains documented as a target-pass reference, not as the current architecture.

## Native path retained and cleaned

The current native path owns the generic Fullhan pipeline directly. Relevant improvements in the cleanup sequence include:

- direct FH8626 platform selection/identity on the OpenIPC ARM1176 target;
- native sensor/ISP/VPU/PAE/VENC startup path retained from the previous migration work;
- corrected ISP runtime statistics-bank, frontend barrier and frame-scheduling semantics retained;
- descriptor copy before release and exactly-once stream release preserved;
- teardown changed to best-effort continuation instead of stopping at the first destructor error;
- force-IDR wired to the recovered PAE `FORCE_I` operation;
- stale provider blockers replaced with blockers that describe current architecture debt;
- CPU/load/memory/uptime/platform/media/capability telemetry added to `/api/status`;
- temperature made explicitly unavailable for FH8626;
- unsafe live MP4 mutation rejected until same-boot native reconfiguration is accepted;
- focused host checks grouped under `tests/fh8626-check.sh`.

## Current capability boundary

Source/reverse-backed current states:

- native H.264 1280x720@25 contract: proven;
- stream lease/release contract: proven;
- GC1054 initialization/order contract: proven;
- direct ISP/kernel bring-up contract: proven;
- rate-control wire mapping used by cold startup: proven;
- force-IDR operation: proven at contract/source level;
- RTX audio transport: hardware-proven independently;
- H.265: unsupported;
- temperature: unsupported/unavailable.

Still unresolved for the latest Divinus candidate:

- fully open sensor/MIPI backend; current startup still uses vendor V100 callback objects;
- complete Divinus audio integration; current capture wrapper expects a separate source-built RTX helper;
- runtime H.264 reconfiguration transaction;
- JPEG/MJPEG target acceptance;
- complete same-boot teardown/restart target acceptance;
- latest-candidate end-to-end hardware acceptance.

Do not reinterpret a source/reverse-proven operation as proof that the latest Divinus binary exercised it successfully on the camera.

## Sensor dependency

`src/hal/full/native/sensor/gc1054/fh8626_sensor_gc1054.c` still loads the V100-era `libmipi.so` and `libgc1054_mipi.so` callback table. The vendor objects are old Fullhan-v2/uClibc ABI material while OpenIPC FH8626 userspace is ARM1176 EABI/musl.

This is retained as a visible bring-up bridge, not a final upstream dependency. Replacement should use the recovered typed MIPI/GC1054 contract and target evidence; do not copy opaque vendor-runtime ownership deeper into Divinus.

## Audio dependency

The accepted board audio path is RTX over `/dev/rtxbus`. The transport itself has hardware evidence, but current Divinus capture still delegates to the separate `fh8626-audio` / `fh8626-audio-rtx` helper. The cleaned Firmware core does not inherently install that old preservation package.

Divinus now checks the helper dependency and fails audio startup explicitly when it is absent. Future cleanup should make the reusable generic RTX transport source-owned in its proper layer while keeping AJL speaker GPIO24 policy outside generic Divinus.

## Runtime configuration policy

The current cold-start H.264 configuration path is the supported hardware-test path. `/api/mp4` writes on FH8626 return not-implemented before mutating configuration because the complete same-boot stop/reconfigure/restart transaction is not yet accepted.

Do not re-enable the generic channel lifecycle for FH8626. A future live configuration implementation should use the recovered native PAE controls and explicit owner lifecycle after target validation.

## Temperature

No FH8626 temperature is exposed. RTC/TSENSOR remain an independent evidence task. Generic `thermal_zone0` presence is not enough to advertise temperature for this board.

## Verification state

`tests/fh8626-check.sh` is the current focused host entry point. It runs:

- `fh8626_contract`;
- `fh8626_hal_stub`;
- `fh8626_native_adapter`;
- `fh8626_native_runtime`;
- `fh8626_provider_boundary`;
- `fh8626_stream_backend`.

The bridge identity cannot write GitHub workflow files, and this API-only pass has no repository shell runner, so these tests were prepared but not executed here. Record their actual result in the next checkout/CI/build pass rather than inferring success.

## Exact next hardware-test sequence

1. Run the focused host suite.
2. Build Firmware `work/fh8626v100-divinus@d589e05...`; confirm it fetched Divinus `168b2ec...` and record resulting binary/image hashes and sizes.
3. Boot/deploy and prove candidate executable/PID/listener ownership.
4. Validate GC1054/ISP startup and sustained 720p25 H.264.
5. Validate raw H.264, RTSP and fMP4 plus reconnect and force-IDR.
6. Validate exposure/color behavior.
7. Validate JPEG/MJPEG only if enabled.
8. Validate RTX audio only with the intended helper/runtime dependency present.
9. Stop and restart Divinus in the same boot and check that media ownership returns cleanly.
10. Validate WIDE/TELE/bootstrap/PTZ/illumination through Builder/device integration.

Fix only failures reproduced against this exact candidate. Do not reopen broad stock reverse unless a concrete failure requires a focused missing contract.

## Provenance cleanup after acceptance

The Firmware direction currently points to the personal fork solely to make the upcoming hardware run attributable. After the Divinus implementation is accepted upstream, restore the package to OpenIPC-owned provenance and remove the staging pin. The candidate is not upstream-ready while vendor sensor/audio helper dependencies remain unresolved.
