# Divinus integration

Divinus is the open reference and diagnostic implementation for FH8626V100. The current source-clean candidate is `ArthurKoba/openipc-divinus/work/fh8626v100@684d0e1fc074435c4d14256c1d5d62ff87c2ebef`.

## Current architecture

The old external `source: fh86` media-owner/socket/FIFO path is retired. Divinus owns the generic FH8626 media pipeline through its native HAL:

`GC1054/MIPI -> ISP -> VPU -> PAE/VENC -> hal_vidstream -> Divinus transports`

Camera-specific board policy does not live in Divinus. GPIO5 cold bootstrap, GPIO4/GPIO14 WIDE/TELE selection, PTZ and illumination remain in the board/device integration that owns them.

## Source cleanup completed

The current work line:

- removed the external encoded-owner source protocol and its transport/wire tests;
- removed the weak board-preparation hook;
- retained and corrected the native ISP/runtime ownership path;
- uses best-effort teardown rather than aborting cleanup on the first destructor error;
- wires recovered native force-IDR;
- exposes platform, CPU/load/memory/uptime, media state and capability telemetry;
- explicitly reports FH8626 temperature as unsupported;
- rejects live `/api/mp4` mutation until the native same-boot reconfigure lifecycle is accepted;
- provides `tests/fh8626-check.sh` for the focused host contract suite.

Historical source-level bank/barrier/frame-wait mismatches are no longer the current candidate description. Their corrected contracts remain part of the source and camera documentation; target behavior still requires fresh validation.

## Remaining native blockers

The provider deliberately remains not production-ready. Current blockers are:

- GC1054/MIPI startup still uses the V100 vendor plug-in callback objects; a fully open native sensor backend remains to be completed;
- RTX transport is hardware-proven, but Divinus currently depends on the separate source-built `fh8626-audio` helper, which the clean Firmware core does not inherently provide;
- full runtime H.264 configuration changes are not accepted; cold-start config is the supported test path;
- same-boot teardown/restart still requires target acceptance;
- the exact latest candidate has not yet completed physical-camera acceptance.

JPEG/MJPEG implementation exists but remains target-unaccepted. H.265 is unsupported in the current FH8626 path.

## Temperature boundary

RTC/TSENSOR remain independent research. Divinus must not manufacture a temperature from `/sys/class/thermal/thermal_zone0` on this board. Current FH8626 status reports temperature unavailable/null.

## Transport constraints

Preserve these established constraints during target debugging:

- capture/source timing drives downstream timestamps;
- one H.264 access unit uses one RTP timestamp;
- reconnect or encoder restart begins a new random-access epoch;
- slow/dead clients must not retain shared media ownership indefinitely;
- every target conclusion must identify the exact candidate executable/PID/listener.

## Exact test staging

Firmware `work/fh8626v100-divinus@3ef425e571f392ea1a2b1cadbeb63cb849ff6bee` temporarily pins Divinus `684d0e1...` so the next image is attributable. That personal-fork pin is staging-only; after upstream acceptance Firmware must return to OpenIPC-owned source provenance.

## Acceptance order

1. host contract suite;
2. exact ARM1176/musl build and artifact identity;
3. GC1054/ISP/native H.264 startup;
4. sustained RTSP/raw H.264/fMP4 and force-IDR/reconnect;
5. ISP exposure/color;
6. optional JPEG/MJPEG;
7. optional RTX audio with the intended helper/runtime present;
8. graceful stop and same-boot restart;
9. board lens/PTZ/illumination integration through the board owner.

Host/source evidence does not promote the candidate to hardware PASS.

## Reverse boundary

If a reproduced Divinus failure needs lower-level analysis, use the canonical Ghidra MCP project for that concrete blocker and promote only the durable result back here. Do not recreate the retired sidecar architecture merely as a workaround.
