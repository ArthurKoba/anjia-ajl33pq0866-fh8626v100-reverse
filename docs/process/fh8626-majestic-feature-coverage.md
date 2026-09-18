# FH8626 Majestic feature coverage

Status: FULL_FEATURE_AUDIT_IN_PROGRESS

This document measures product-facing Majestic feature coverage, not merely the
ability to start the media stack. A feature is COMPLETE only when the Majestic
public API surface is implemented or deliberately delegated to a compatible
vendor layer and the required runtime ownership is understood.

## Current estimate

Approximate offline feature coverage: 75-80%.

This is intentionally lower than the earlier "offline port complete" label.
That earlier label referred to the minimum source compatibility path needed to
reach a real target run. It did not mean every Majestic setting/API was covered.

## Video codecs and channels

| Area | Status | Notes |
|---|---|---|
| H.264 channel 0 create/start/stop | IMPLEMENTED | Native FH8626 PAE path |
| H.264 profile | IMPLEMENTED | Baseline/Main public values recovered |
| H.264 visible resolution | IMPLEMENTED | Public SetChnAttr now translated instead of fixed 720p |
| H.264 GOP/key interval | IMPLEMENTED | Taken from public SetChnAttr |
| H.264 initial RC modes | IMPLEMENTED/PENDING_REVIEW | Apollo translator recovered for public modes 3/4/5/6/0xB -> native RC modes |
| H.264 realtime RC change | IMPLEMENTED | Six public words + channel -> native 0x1c realtime RC request |
| H.264 full SetRCAttr/GetRCAttr | PARTIAL | Native 0x54 requests are known; standalone public union translation still being completed |
| H.264 IDR | IMPLEMENTED | Native force-I |
| H.264 entropy/deblock/slice/intra-refresh | NOT_IMPLEMENTED | Exports currently loader stubs |
| ROI/background QP/lost-frame/de-breath | NOT_IMPLEMENTED | Exports currently loader stubs |
| rotate | NOT_IMPLEMENTED | VENC/VPSS rotate exports are stubs |
| second/sub stream | NOT_COMPLETE | Current source path is channel-0-centric; VPU/VENC bind, PAE memory and stream lease ownership need per-channel implementation |
| H.265/HEVC | NOT_IMPLEMENTED | Apollo contains /dev/hevc userspace backend but current source facade leaves H.265 controls as stubs |
| JPEG/MJPEG snapshot | NOT_IMPLEMENTED | Native JPEG driver is reversed, but Majestic-facing _JPEG_* facade is still loader stubs |

## VPSS / image path

Implemented native translations:
- system/channel memory;
- VI attributes;
- channel geometry/open/close;
- enable/global disable;
- exact frame-control wire;
- freeze/unfreeze compatibility surface;
- media bind for the currently implemented video path.

Still stubs or incomplete:
- crop;
- mask;
- OSD;
- OSD highlight/invert;
- RGB pre-processing;
- graph APIs;
- scaler coefficients/default scaler size;
- APC channel controls;
- YC mean/statistics;
- low-latency controls;
- user-picture/frame-buffer APIs;
- advanced frame extraction/lock APIs.

These must be prioritized by actual Majestic imports/call paths, not by the
size of the vendor SDK export list.

## ISP and image controls

The donor ISP/ispcore layer is intentionally retained while compatible because
stock Apollo proves that many API_ISP_* functions mutate a large shared
userspace context rather than map one-to-one to ioctls.

Recovered/understood public controls include:
- mirror/flip and Bayer-aware mirror/flip-ex;
- LTM;
- contrast;
- saturation;
- APC;
- sensor format / VI attributes;
- AE/AWB related control families.

This is not yet equivalent to a complete source ISP replacement. Product
acceptance requires proving the retained donor ISP path after the sensor/MIPI/
VMM/VPSS corrections, then replacing only concrete incompatible surfaces.

## Audio

Implemented:
- RTX transport init/reset/mmap;
- seven-word config transport;
- AI enable/disable;
- AO enable/disable;
- capture frame + PTS;
- playback frame;
- AI/AO volume;
- buffer clear / playback wait;
- balanced teardown.

Incomplete:
- automatic board DSP init payload ownership inside FH_AC_Init;
- full AEC;
- AGC;
- capture NR;
- playback NR;
- pause/resume/full-duplex policy;
- complete two-way-audio product ownership.

## Analytics / statistics / newer Majestic features

Not yet claimed complete:
- motion/analytics data paths;
- BGM integration;
- NN bind path;
- YC/statistics APIs;
- auxiliary Y/YUV frame requests;
- hardware average-time/status APIs;
- any newer Majestic telemetry that depends on platform-specific provider
  callbacks;
- the historical /metrics backend/provider surface.

Majestic HTTP/WebUI itself remains untouched. Missing platform telemetry must be
implemented at the real provider/API boundary, not hidden by a proxy or
frontend patch.

## Configuration surface that must work before product-complete

At minimum, the final implementation must support and regress:
- main and sub stream enable/disable;
- supported codec selection;
- per-stream width/height;
- fps/frame-control;
- bitrate;
- RC mode;
- GOP;
- profile;
- runtime bitrate change;
- IDR;
- JPEG quality/fps/snapshot;
- image mirror/flip/rotation where advertised;
- day/night/image controls;
- microphone and speaker/two-way audio;
- OSD where Majestic advertises it;
- motion/analytics/statistics where Majestic advertises them;
- repeated stop/start/reconfigure without reboot.

## Rule for completion

Do not mark FULL_FEATURE_COMPLETE because all dynamic symbols resolve. Loader
stubs only prove loadability.

A feature is complete only when one of these is true:
1. source implementation is recovered and wired;
2. retained donor implementation is shown ABI-compatible with FH8626;
3. Majestic does not use/expose the function for this platform and the omission
   is explicitly documented.

The current priority order is:
1. full H.264 RC + channel-attribute round-trip;
2. multi-stream ownership;
3. JPEG;
4. H.265 if the current Majestic build exposes it on this platform;
5. image/OSD/ROI/crop controls actually used by Majestic;
6. analytics/statistics provider surface;
7. full audio DSP policy.
