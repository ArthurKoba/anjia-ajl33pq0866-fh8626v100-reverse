# FH8626 Majestic feature coverage

Status: FULL_FEATURE_OFFLINE_CLOSURE / BUILD_AND_HARDWARE_PENDING

This document measures product-facing Majestic feature coverage, not merely
loader compatibility.

## Current estimate

Approximate offline coverage for the current FH8852V200 Lite Majestic feature
surface on FH8626V100: **~97%**.

The remaining percentage is not core H.264/JPEG/audio functionality. It is
mainly unobserved or SDK-only controls such as crop/rotate/slice variants,
capability advertisement for hardware-unsupported modes, and target-only
lifecycle validation.

Canonical cross-agent coordination: reverse issue #3.

Current refs:
- Firmware Majestic: `work/fh8626v100-majestic@a60789ae`
- Divinus: `work/fh8626v100@5979e160`

## Video

Implemented source/native compatibility:
- H.264 channel create/start/stop for the product main/sub/analytics channel
  topology;
- exact VPU->VENC bind mapping `source=vpu+1`, `destination=venc+7`;
- channel-aware shared H.264 FIFO routing using native descriptor channel
  `desc[3]`, with query=peek and release=pop ownership;
- visible width/height, Baseline/Main profile, GOP/key interval;
- normal and smart H.264 public channel attributes;
- full VBR/CBR/fixed-QP/AVBR/CVBR public->native RC translation;
- full RC set/get and channel-attribute readback;
- realtime RC change through native 0x1c request;
- IDR/force-I;
- stop/restart state ownership without toggling global ISP producer state.

H.265/HEVC is **hardware/driver unsupported**, not an unfinished shim. Stock
FH8626 `enc.ko` registers only H.264 media stream kind 4 and no HEVC encoder
engine/module. H.265-specific SDK calls return an explicit unsupported result.

Remaining video controls requiring evidence before implementation:
- crop;
- VENC/VPSS rotation beyond paths already handled by retained ISP/image layer;
- explicit H.264 slice split / entropy / deblock / intra-refresh SDK calls;
- ROI/background-QP/de-breath SDK extras.

These remaining exports are not reachable from the current selected runtime
closure. A moving Majestic direct import of one now fails the build.

## JPEG / MJPEG

Implemented Majestic-facing source surface:
- system init;
- memory query/create/destroy;
- snapshot JPEG and continuous MJPEG channel config/readback;
- stock JPEG hardware quality LUT;
- MJPEG RC set/get;
- start/stop;
- rotate;
- generic stream query;
- raw->public stream conversion and balanced release;
- frame submit / submit-ex;
- MJPEG drop policy;
- hardware timing projection.

Cross-agent note: Divinus app-facing quality accepts 1..99 percent and converts
to a 0..9 hardware bucket. FH8852 public `_JPEG_SetChnAttr` already carries
the 0..9 bucket. These are intentionally different layers over the same native
quality LUT.

## Motion / statistics / OSD

Optional donor feature backends are restored:
- `libadvapi_md.so`;
- `libadvapi_osd.so`.

Their required FH8626-facing VPSS dependencies are implemented:
- GetViAttr;
- GetChnAttr;
- YC mean enable/disable/mode;
- GetYCmean;
- GetCPYData;
- channel/global GraphV2 get/set.

GraphV2 uses the recovered FH8626 folded 0x448-byte request
`0xC448696D/0xC448696E`, translating the FH8852 public 273-word GraphV2
object.

BGM/NN SDK exports that are not selected or imported by the current runtime
remain outside the advertised closure rather than returning fake success.

## ISP / image

Donor `libisp.so`, `libispcore.so` and `libadvapi_isp.so` remain
intentionally isolated as one coherent userspace ISP context implementation.
This is deliberate: stock Apollo shows that AE/AWB/LTM/contrast/saturation/APC
and mirror/flip operations mutate a large shared userspace ISP context and are
not simple one-ioctl calls.

Current direct Majestic requirement `FHAdv_Isp_SetColorMode` is provided by
that retained coherent layer. Replacing isolated functions with partial source
shims would reduce correctness until the complete context state machine is
reimplemented.

## Audio

The recovered Majestic ACW/RTX surface is source implemented:
- transport reset/init/mmap/deinit;
- exact AJL33PQ0866 retail 0x17a DSP init payload applied during `FH_AC_Init`;
- 7-word config and selector extension;
- AI/AO enable/disable;
- capture frame / PTS and raw-frame paths;
- playback frame;
- analog/digital/channel volume controls;
- clear/wait/pause/resume/sync;
- AEC, capture NR, playback NR and AGC config commands;
- HPF/mic-bias extension controls;
- variable `Ext2` ioctl;
- external-codec init surface.

No known FH8852 ACW loader stub remains in the recovered Majestic surface.

## ABI closure guards

The package now verifies at build time:
1. every direct Fullhan import of the moving Majestic binary has a selected
   provider;
2. a direct import may not resolve only to a known unsupported compatibility
   stub;
3. selected donor feature libraries are also checked transitively, so
   motion/OSD/ISP library updates cannot silently begin depending on an
   unsupported FH8626 API.

At the current refs, selected donor libraries have **zero** dependencies on the
remaining unsupported compatibility exports.

## Web/control plane

Majestic HTTP, API, WebSocket and WebUI remain untouched. No proxy and no
frontend patch is part of this port.

The historical `/metrics` symptom is a separate Majestic/platform-provider
question and must not be hidden by auxiliary HTTP code.

## Remaining offline queue

Before declaring every advertised setting complete:
1. determine whether the current Fullhan Majestic build actually invokes
   crop/rotate/slice SDK controls (including dynamic lookup paths);
2. ensure hardware-unsupported H.265/high-profile options are not falsely
   advertised by the target capability schema;
3. keep the Divinus and Majestic RC/JPEG/audio wire definitions synchronized
   through reverse issue #3;
4. then stop offline work and move to build/flash/target validation.

Target acceptance must cover:
- main/sub enable/disable and simultaneous streams;
- bitrate/RC/GOP/profile/readback and live bitrate changes;
- JPEG snapshot and MJPEG;
- OSD/privacy graph;
- motion/statistics;
- image/day-night controls;
- microphone, speaker and audio VQE;
- repeated stop/start/reconfigure without reboot.
