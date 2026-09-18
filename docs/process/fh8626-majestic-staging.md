# FH8626V100 Majestic staging

Status: `FULL_FEATURE_OFFLINE_CLOSURE / BUILD_GATE_READY / HARDWARE_PENDING`

Checked: 2026-09-18.

This document is the camera-level coordination authority for the FH8626V100
Majestic direction. The active product experiment runs the FH8852V200 Majestic
userspace on FH8626V100 through a source compatibility boundary plus a retained,
pinned FH8626 kernel/ARC media runtime.

This remains a compatibility/bring-up architecture. It is not a claim that
Majestic officially supports FH8626V100.

## Current repository checkpoint

- Firmware core: `ArthurKoba/openipc-firmware/work/fh8626v100@80169887`
- Firmware Majestic: `ArthurKoba/openipc-firmware/work/fh8626v100-majestic@04e09360`
- Firmware Divinus composition: `ArthurKoba/openipc-firmware/work/fh8626v100-divinus@255b8c8d`
- Builder: `ArthurKoba/openipc-builder/work/fh8626v100-anjia@ee0687c0`
- Linux staging: `ArthurKoba/openipc-linux/work/fh8626v100@357c2d13`
- Divinus peer implementation: `ArthurKoba/openipc-divinus/work/fh8626v100@44c4fb94`
- U-Boot native direction: `ArthurKoba/u-boot-fullhan/fh8626v100-mainline@7ac0aa7e`
- Cross-agent coordination: reverse issue #3.

Exact build provenance must record full resolved SHAs rather than relying on
these shortened documentation locators.

## Historical hardware evidence

The retained historical experiment proves only the following facts:

1. the FH8852V200 ARM/musl Majestic executable can load on the real
   FH8626V100 when its userspace dependencies are supplied;
2. Majestic can reach ordinary process startup and serve its own HTTP/WebUI
   control plane on port 80;
3. media-off startup avoids `start_sdk()` and was stable enough to prove the
   control plane;
4. enabling the old media path reached the explicitly selected GC1054 path and
   then failed inside SDK startup;
5. direct sensor reads still returned GC1054 ID `0x10/0x54`.

That old crash is no longer described as an unexplained sensor failure. Reverse
analysis proved that the FH8852 sensor callback table is 0x7c bytes and the
native FH8626 GC1054 callback table is 0x68 bytes with a different callback
order. Feeding the native object directly to an FH8852 consumer was an ABI
error.

Historical control-plane evidence is not evidence that the current source
compatibility layer, current kernel, current rootfs composition or current full
feature profile has passed hardware.

## Accepted architecture

Majestic HTTP/API/WebSocket/WebUI remain native. Port 80 belongs directly to
Majestic. No JavaScript patch, route interception or auxiliary HTTP proxy is
part of the accepted architecture.

The stack is:

`Majestic FH8852 public ABI -> FH8626 compatibility source -> native FH8626 ABI`

where the native ABI is established by Apollo/userspace/kernel Ghidra evidence,
not by assuming Divinus is correct.

Divinus is a peer implementation and regression oracle. Shared platform
contracts are synchronized through issue #3. If Divinus and Majestic disagree,
stock/Ghidra evidence decides.

## Shared FH8626 kernel/ARC media runtime

Source cleanup correctly removed opaque media payloads from active branches,
but a pre-deploy audit found that those still-required bytes temporarily had no
build-time owner. This is fixed by Firmware package
`fullhan-media-fh8626v100`.

The package downloads the exact hardware-proven payloads from immutable
preservation commit:

`f4bf49da6ef355c9e733e00d774efe403513b1d4`

and verifies each payload by SHA-256:

- `vmm.ko`
- `xbus_rpc.ko`
- `media_process.ko`
- `isp.ko`
- `enc.ko`
- `jpeg.ko`
- `bgm.ko`
- `gpio_wave.ko`
- `rtthread_arc.bin`

Active source branches do not store these binaries.

The current Majestic compatibility package explicitly selects
`BR2_PACKAGE_FULLHAN_MEDIA_FH8626V100`, so selecting Majestic cannot silently
produce a media image without the required FH8626 kernel/ARC runtime.

OpenIPC `S70vendor` invokes `load_fullhan -i`. The loader preserves the
recovered order:

`VMM -> XBUS/ARC -> media_process -> ISP -> enc -> jpeg -> bgm -> gpio_wave`

and requires the critical media device nodes:

- `/dev/vmm_userdev`
- `/dev/media_process`
- `/dev/isp`
- `/dev/pae`
- `/dev/jpeg`

The retained modules report
`vermagic=4.9.129 mod_unload ARMv6 p2v8`, matching the target kernel family.
That is a static compatibility result; actual `insmod` remains a hardware
gate.

## Current Majestic userspace compatibility closure

The active FH8852-facing ABI is no longer an eight-library donor stack.

Source-built compatibility now owns:

- GC1054 FH8852-shaped sensor facade over the recovered FH8626 sensor contract;
- active GC1054 native implementation;
- MIPI;
- VMM;
- SYS/VPSS;
- multi-channel H.264 VENC/RC;
- encoded stream acquire/release and ring-wrap translation;
- JPEG/MJPEG;
- motion-facing YC mean/CPY VPSS surface;
- OSD-facing GraphV2 VPSS surface;
- ACW/RTX audio and VQE-facing calls.

The retained donor userspace closure is intentionally limited to:

- `libadvapi.so`
- `libadvapi_isp.so`
- `libadvapi_md.so`
- `libadvapi_osd.so`
- `libadvapi_smartir.so`
- `libisp.so`
- `libispcore.so`

The donor ISP/ispcore/advapi layer remains coherent because stock Apollo shows a
large shared userspace ISP context/state machine. Replacing isolated functions
with partial guessed shims would be less correct than retaining that coherent
layer until a demonstrated incompatibility requires replacement.

No FH8852 kernel modules, FH8852 ARC firmware, donor load scripts or donor
sensor plug-ins are installed.

## Sensor / MIPI

The original sensor failure is closed architecturally:

- FH8852 public sensor callback object: 0x7c bytes;
- FH8626 native GC1054 callback object: 0x68 bytes;
- callback order differs;
- the compatibility facade presents the FH8852 shape and translates recovered
  operations onto the FH8626 sensor contract.

The active GC1054/MIPI path is source-owned. Archived vendor GC1054/MIPI blobs
must not be reintroduced as fallback.

## VPSS / VENC / stream

Ghidra/kernel reverse established the native lifecycle and corrected earlier
Divinus-derived assumptions:

- `FH_VPSS_Enable(channel)` carries the channel id, not boolean 1;
- VPU disable is separate request `0xC004694E`;
- OpenChn is `0xC004694F`;
- CloseChn is `0xC0046950`;
- frame control is an 8-byte `{channel, packed_fps}` request;
- stock sequencing separates VI/VPU ownership from encoder lifecycle.

The H.264 compatibility path supports the current product surface:

- main/sub/analytics channel topology;
- capacity-aware CreateChn;
- resolution/profile/GOP;
- VBR, CBR, AVBR, CVBR and fixed-QP style RC;
- RC/channel readback;
- realtime RC changes;
- IDR;
- correct VPU -> VENC bind ids;
- shared encoded FIFO routing by native descriptor channel;
- bounded stream acquire;
- ring wrap conversion;
- balanced exactly-once release.

The source layer does not own a fake global producer gate per encoder channel.

H.265/HEVC is explicitly unsupported because the retained FH8626 stock encoder
stack does not provide an HEVC engine. Unsupported H.265 calls return an error
rather than fake success.

## JPEG / MJPEG

The Majestic-facing source layer covers:

- memory query/init/uninit;
- snapshot JPEG and continuous MJPEG;
- channel configuration/readback;
- native quality mapping;
- MJPEG RC/readback;
- start/stop;
- stream query/translation/release;
- rotate;
- submit/submit-ex;
- drop policy;
- hardware timing projection.

The Divinus and Majestic quality APIs intentionally differ by layer: Divinus
accepts application percentage and converts to a hardware bucket; the FH8852
public JPEG ABI already supplies the bucket.

## Motion / OSD / GraphV2

Motion and OSD retain their small donor feature backends
`libadvapi_md.so`/`libadvapi_osd.so`, while the FH8626-facing VPSS operations
they require are source translated.

Native GraphV2 is the folded 0x448-byte request family
`0xC448696D/0xC448696E`, translating the FH8852 public 273-word object into
one selector + 273-word native record.

A final cross-agent audit found a missing validation boundary in the Majestic
facade. Divinus had introduced native GraphV2 slot limits; this was then
verified independently in Ghidra against `isp.ko:vpu_set_logov2`:

- selector/logov2-number: 0..2;
- global selector 0 graph index: 0..1;
- channel selectors 1/2 graph index: 0..3.

Firmware Majestic `04e09360` now enforces those limits before issuing the
native ioctl. This is a shared native contract, not a Divinus-specific policy.

## Audio

The Majestic ACW/RTX facade includes:

- transport init/reset/mmap/deinit;
- exact AJL33PQ0866 retail DSP init payload in normal `FH_AC_Init`;
- separate external-codec init behavior;
- 7-word config and selector extension;
- AI/AO enable/disable;
- capture frames, raw paths and PTS;
- playback;
- volume controls;
- pause/resume/clear/wait/sync;
- AEC, AGC, capture NR and playback NR-facing commands;
- HPF/mic-bias/extension controls.

Physical speaker-amplifier GPIO24 policy remains board-owned. The generic
audio facade uses a board-neutral lifecycle hook: unmute after AO is ready,
mute before AO teardown.

## Day/night

The full Majestic profile uses the recovered ANJIA board contract:

- IR-cut DAY/closed coil: GPIO18;
- IR-cut NIGHT/open coil: GPIO60;
- bistable pulse: 190 ms;
- IR LED: GPIO25 active high;
- white LED: GPIO23 active high, shared with SADC1;
- ambient light: SADC1;
- speaker mute: GPIO24 active high.

The full profile maps the IR-cut pair and IR backlight into Majestic's native
day/night configuration. The white LED is not pretended to be a second
Majestic backlight.

Physical direction still requires target acceptance. A reversed result must be
debugged against the board contract rather than hidden by undocumented pin
swaps.

## ABI guards

The package checks:

1. direct Fullhan imports of the downloaded Majestic binary;
2. whether a required symbol resolves only to an unsupported stub;
3. transitive imports of the retained donor feature libraries.

At the current selected closure, retained donor libraries have no dependency on
remaining unsupported compatibility exports.

This means a future moving Majestic build that begins importing a new unsupported
SDK symbol should fail the build instead of silently producing a partial image.

## Moving Majestic donor boundary

The FH8626 kernel/ARC payload is immutable and SHA-256 pinned.

The FH8852V200 Majestic executable itself is still downloaded from the upstream
moving object:

`majestic.fh8852v200.lite.master.tar.bz2`

The first controlled build must therefore retain the exact downloaded/installed
Majestic SHA-256 and the resolved build provenance.

The direct/transitive ABI guard reduces the risk of an unnoticed API expansion,
but a moving donor is not full product reproducibility. Production acceptance
still requires an immutable donor object, official FH8626 Majestic support, or
an otherwise reproducible and fully characterized binary contract.

This does not prevent a controlled first hardware bring-up when exact bytes are
recorded.

## Builder assembly

The active Builder has one ANJIA device tree and composes runtime variants
rather than maintaining a separate Majestic branch.

Majestic target:

`fh8626v100_lite_anjia-ajl33pq0866_majestic`

Composition:

1. Firmware `br-ext-chip-fullhan/configs/fh8626v100_lite_defconfig`;
2. Builder ANJIA `base.config`;
3. short Majestic runtime fragment.

The sibling `.firmware` metadata selects
`ArthurKoba/openipc-firmware@work/fh8626v100-majestic` automatically.

Normal invocation is:

```sh
./builder.sh fh8626v100_lite_anjia-ajl33pq0866_majestic
```

Environment repo/ref overrides are for explicit bisect/debug only.

## Default boot and acceptance runners

Default boot remains media-off. That keeps the historical control-plane baseline
separate from full media acceptance.

The image supplies a diagnostic ladder ending in
`majestic-fh8626-full-run`, whose strict profile enables native media with
permissive stubs disabled.

The full profile exercises together:

- H.264 main 1280x720@25;
- H.264 sub 640x360@25;
- JPEG 640x384;
- OSD;
- motion;
- audio input/output;
- RTSP;
- ANJIA day/night.

It does not replace the persistent media-off configuration.

## Image limits and first build gate

The U-Boot/Firmware layout requires:

- `uImage <= 2048 KiB`
- `rootfs.squashfs <= 5120 KiB`

The nine proprietary media/ARC payloads account for roughly 798 KiB raw before
SquashFS, so final rootfs headroom must be measured rather than assumed.

The active branches are still `NOT_BUILT` as a composed current image. No
historical image size is accepted as proof for the current tree.

## First hardware ladder

After a successful exact build:

1. retain resolved Builder/Firmware/Linux SHAs and build config;
2. retain final image sizes/hashes and media-package hash verification;
3. record installed Majestic SHA-256;
4. boot using the already-working boot chain first; do not mix the initial
   Majestic media experiment with U-Boot migration;
5. confirm `S70vendor` loads the FH8626 media runtime and required device
   nodes exist;
6. prove the unchanged media-off Majestic baseline;
7. run `majestic-fh8626-abi-probe`;
8. use narrower strict/native runners only to localize a failure;
9. run `majestic-fh8626-full-run`;
10. prove simultaneous main/sub RTSP;
11. exercise bitrate/RC/GOP/profile/readback and live reconfiguration;
12. test JPEG/MJPEG, OSD, motion, image/day-night, microphone, speaker/talkback;
13. regress repeated stop/start/restart without reboot;
14. regress PTZ, lens/bootstrap, illumination, storage, Wi-Fi and shutdown.

Only failures reproduced on the exact built candidate should drive new
compatibility code.

## U-Boot boundary

Builder does not build or flash U-Boot.

The native OpenIPC U-Boot direction is separately source/build accepted at
`fh8626v100-mainline@7ac0aa7e`, but full migration is not yet hardware
accepted. Keep it out of the first Majestic userspace/media test. After media
acceptance, U-Boot migration is a separate cold-boot/recovery gate with a full
flash backup and external SPI recovery available.

## Completion definition

For the currently demonstrated FH8852V200 Lite runtime surface, offline source
coverage is considered complete.

The next phase is not more speculative SDK emulation. New code should be driven
by one of:

- a real build failure;
- ABI guard failure on the exact downloaded Majestic object;
- target runtime failure;
- new Ghidra evidence that a currently unsupported API is actually required.

Production readiness remains stricter than a successful first video stream. It
requires reproducible build/runtime contracts and target evidence for the full
selected feature surface.
