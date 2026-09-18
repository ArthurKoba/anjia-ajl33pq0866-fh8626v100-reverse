# FH8626 Majestic feature coverage

Status: FULL_FEATURE_OFFLINE_CLOSURE / BUILD_GATE_READY / HARDWARE_PENDING

This document measures product-facing Majestic feature coverage, not merely
loader compatibility.

## Current estimate

Approximate offline coverage for the current FH8852V200 Lite Majestic feature
surface on FH8626V100: **100% of the currently demonstrated FH8852V200 Lite runtime closure, offline**.

The offline implementation percentage is now closed for the **currently
demonstrated and selected FH8852V200 Lite runtime closure**. SDK-only exports
that are not imported or reachable from the selected runtime are not counted
as missing product functionality; they remain explicit unsupported boundaries
and are protected by build-time direct + transitive ABI guards.

This does **not** mean target acceptance is complete. Build, flash and hardware
validation remain a separate next phase.

Canonical cross-agent coordination: reverse issue #3.

Current refs:
- Firmware Majestic: `work/fh8626v100-majestic@04e09360`
- Divinus: `work/fh8626v100@44c4fb94`
- Builder: `work/fh8626v100-anjia@a02d325e`
- Linux: `work/fh8626v100@357c2d13`

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
object. A final Divinus cross-check was independently verified against Ghidra
`isp.ko:vpu_set_logov2`: selector is 0..2, global graph index is 0..1 and
channel graph index is 0..3. Majestic now rejects out-of-range slots before
the native ioctl.

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

## Runtime packaging and reproducibility

The compatibility package explicitly selects the shared
`fullhan-media-fh8626v100` runtime. Its eight FH8626 media modules plus
`rtthread_arc.bin` are fetched from immutable Firmware archive commit
`f4bf49da6ef355c9e733e00d774efe403513b1d4` and each file is SHA-256
verified. No FH8852 kernel/ARC payload is installed.

The FH8852V200 Majestic executable remains a separate supply-chain boundary:
the package still downloads `majestic.fh8852v200.lite.master.tar.bz2` from
the moving upstream object. ABI guards catch an unsupported import expansion,
but the first controlled build must also record the exact installed Majestic
SHA-256. An immutable donor or official FH8626 build remains required for full
product reproducibility.

## Web/control plane

Majestic HTTP, API, WebSocket and WebUI remain untouched. No proxy and no
frontend patch is part of this port.

The historical `/metrics` symptom is a separate Majestic/platform-provider
question and must not be hidden by auxiliary HTTP code.

## Strict full-feature acceptance profile

Firmware now installs `majestic-fh8626-full-run`. It selects
`native-strict`: native FH8626 VENC is enabled while permissive
`FH8626_MAJESTIC_STUB_OK` remains disabled.

The dedicated full profile simultaneously enables:
- H.264 main 1280x720@25;
- H.264 sub 640x360@25;
- JPEG 640x384 at 5 fps;
- stock-schema OSD with the selected `majestic-fonts` asset;
- motion detection;
- 8 kHz Opus capture plus audio output;
- RTSP.

It is intentionally separate from the persistent media-off default service.

## Remaining offline queue

Only unobserved SDK-only controls remain:
1. crop/extra rotate/slice controls if the current Fullhan Majestic build is
   later proved to import or dynamically call them;
2. capability advertisement for H.265/high-profile must remain honest about
   FH8626 hardware support;
3. keep Majestic and Divinus contracts synchronized through reverse issue #3.

The current historical FH8852 build/report contains no evidence of crop,
sliceUnits or HEVC use, and the selected donor closure has zero dependencies on
the remaining unsupported exports. A build-time guard now rejects future
Majestic/vendor updates that cross that boundary.

There is no usable current CI build for these fork-local staging branches, so
the exact composed Buildroot compile is the next owner gate. Current hard
limits remain `uImage <= 2048 KiB` and `rootfs.squashfs <= 5120 KiB`.

Target acceptance must cover:
- main/sub enable/disable and simultaneous streams;
- bitrate/RC/GOP/profile/readback and live bitrate changes;
- JPEG snapshot and MJPEG;
- OSD/privacy graph;
- motion/statistics;
- image/day-night controls;
- microphone, speaker and audio VQE;
- repeated stop/start/reconfigure without reboot.


## Offline closure reached

The offline closure is now considered complete for the current target/runtime:

- selected direct + transitive Fullhan ABI has no reachable unsupported export;
- strict full-feature runner covers main/sub H.264, JPEG, OSD, motion, audio,
  RTSP and board day/night wiring without permissive stubs;
- ANJIA speaker-amplifier mute is integrated via a board-owned AO lifecycle
  hook, not embedded GPIO policy;
- ANJIA day/night full profile carries the recovered GPIO18/GPIO60 IR-cut pair,
  GPIO25 IR illumination and the hardware-proven 190 ms pulse;
- source replacements no longer ship donor libdsp/libmipi/libvmm fallbacks;
- OSD font assets are selected explicitly;
- Divinus and Majestic contract differences are tracked through issue #3 and
  current Divinus already includes the major VPSS lifecycle corrections.

From this point, new code should be driven by either:
1. a build failure;
2. a target/runtime failure;
3. a moving Majestic/vendor update rejected by the ABI guard;
4. new Ghidra evidence proving a currently unsupported advertised feature is
   actually part of this target's runtime contract.

Until one of those occurs, further SDK emulation would be speculative rather
than completion work.
