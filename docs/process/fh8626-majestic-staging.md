# FH8626V100 Majestic staging

Status: `STAGED / HISTORICAL_CONTROL_PLANE_HARDWARE_PASS / SENSOR_ABI_ADAPTER_READY / NEW_BUILD_PENDING`.

Checked: 2026-09-18.

This document is the coordination authority for the FH8626V100 Majestic direction. The goal of the current branch is **not** to finish the FH8626 media port. It preserves the proven Majestic control-plane experiment in a cleaner build architecture so Builder can reproduce it on top of the current FH8626 core.

## Historical experiment

The historical Builder experiment is identified by immutable Git SHAs only:

- WIP commit: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`;
- parent/checkpoint: `5603a701c8812aebc705c42e933ebae48aed805f`;
- commit message: `WIP: preserve FH8626V100 Majestic builder experiment`.

No live Builder branch or tag is retained for that experiment. Its useful content has been reconstructed on the current clean staging branches.

That one commit mixed useful Majestic work with obsolete platform material. The copied generic FH8626 kernel config and the 2993-line Firmware-side kernel patch are **not** carried forward: the current Firmware/Linux core already owns those concerns cleanly.

The useful historical conclusions are:

1. the FH8852V200 Majestic ARM/musl binary loads on the real FH8626V100 when its userspace dependency set is supplied;
2. the loader dependency closure was reduced to eight FH8852V200 vendor libraries: `libadvapi.so`, `libadvapi_isp.so`, `libadvapi_smartir.so`, `libdsp.so`, `libisp.so`, `libispcore.so`, `libmipi.so`, `libvmm.so`;
3. Majestic reached normal process startup and served HTTP on port 80;
4. sensor autodetection did not work; with the native GC1054 plug-in explicitly selected, SDK initialization reached the sensor path and then segfaulted;
5. direct I2C reads on bus 0 returned GC1054 chip ID bytes `0x10` and `0x54`, so that failure did not establish a dead/reset sensor;
6. disabling media consumers prevented `start_sdk()` from being entered and left Majestic running successfully as an HTTP/control plane;
7. a later image proved automatic `S95majestic` startup, built-in haserl, WebUI files under `/var/www`, and Majestic listening on port 80.

The old image was constrained by the historical 3776 KiB rootfs partition and had to be trimmed to about 3694 KiB. The current core targets the standard OpenIPC 5120 KiB rootfs partition; the historical size workaround is evidence, not a reason to restore the old partition layout.

## Current branch architecture

Firmware:

- shared streamer-neutral core: `ArthurKoba/openipc-firmware/work/fh8626v100@eabd1ccd4684af6997771269c4655f7e4435bcec`;
- Divinus direction: current dedicated `work/fh8626v100-divinus` line;
- Majestic direction: current dedicated `work/fh8626v100-majestic` line (latest ref is tracked in `STATE.md`).

Builder:

- single active ANJIA device line: `ArthurKoba/openipc-builder/work/fh8626v100-anjia@9c507b85481df2787bb214e535f17003bdebb6e6`;
- composed Majestic target: `fh8626v100_lite_anjia-ajl33pq0866_majestic`;
- Majestic fragment: `br-ext-chip-fullhan/configs/variants/fh8626v100_lite_anjia-ajl33pq0866_majestic.config`;
- matching Firmware metadata: `br-ext-chip-fullhan/configs/variants/fh8626v100_lite_anjia-ajl33pq0866_majestic.firmware`;
- retired separate Builder Majestic branch is preserved only as `archive/fh8626v100-anjia-majestic-branch-20260918`.

Builder no longer carries a copied Majestic device profile. It composes the Firmware generic FH8626 lite defconfig, one shared ANJIA board delta and the short Majestic runtime fragment. The fragment selects only the Firmware-owned Majestic compatibility package; GPIO, Wi-Fi, microSD, illumination, lens, persistent board policy and optional PTZ come from the same ANJIA base/package used by the other variants.

Core platform/kernel fixes belong in shared Firmware/Linux. Majestic compatibility implementation remains in the Majestic Firmware direction unless/until ownership changes upstream.


## Reconstructed Majestic Firmware integration

The Majestic Firmware branch adds only:

- one Buildroot package, `majestic-fh8852v200-compat`;
- the package's Config.in registration;
- Majestic selection in the branch's FH8626 lite defconfig.

The package:

- downloads the FH8852V200 Majestic `lite.master` binary;
- stores the donor executable outside the normal PATH and exposes `/usr/bin/majestic` through a wrapper;
- isolates the eight donor libraries under `/usr/lib/majestic-fh8852v200`;
- reuses the normal OpenIPC json-c/libevent/libogg/libyaml/mbedTLS/Opus packages;
- selects Majestic WebUI/haserl;
- installs a media-off `/etc/majestic.yaml`;
- installs an init script using the hardware-proven HTTP-only invocation;
- builds and installs `majestic-fh8626-abi-probe`, which performs no SDK/media initialization and makes no device writes;
- builds a source `majestic-fh8626/libgc1054_mipi.so` facade plus an explicit `majestic-fh8626-media-run` bring-up runner;
- keeps the normal init service media-off; the sensor facade is not selected by default;
- installs **no** FH8852 kernel modules, firmware, load scripts or donor sensor plug-ins.

This is intentionally a compatibility staging package, not a statement that FH8852 userspace blobs are the final FH8626 media architecture.

## FH8852 API to FH8626 native translation map

Static inspection of the exact eight donor libraries already present in Firmware, combined with the recovered FH8626 contracts used by Divinus, gives the following implementation map. Symbol presence is donor-library evidence; the FH8626 side comes from the platform reverse/native implementation. It does **not** prove that FH8852 and FH8626 structures have identical layouts.

| FH8852-facing API family | Representative donor symbols | FH8626 native operation |
|---|---|---|
| VMM | `FH_SYS_VmmAlloc`, `FH_SYS_VmmFree`, `FH_SYS_Mmap`, `FH_SYS_Munmap` | `/dev/vmm_userdev` and recovered VMM allocation/mapping contract |
| system/media bind | `FH_SYS_Init`, `FH_SYS_Exit`, `FH_SYS_BindVpu2Enc` | native media ownership, `MEDIA_BIND` / unbind and lifecycle ledger |
| VI/VPSS/VPU | `FH_VPSS_SysInitMem`, `FH_VPSS_SetViAttr`, `Query*Mem`, `ChnInitMem`, `OpenChn`, `Enable` | recovered VPU system/channel memory, VI attributes, channel open/enable and frame-control ioctls |
| H.264 VENC | `FH_VENC_SysInitMem`, `CreateChn`, `SetChnAttr`, `StartRecvPic` | recovered PAE system/channel memory, encoder config and start |
| encoded stream | `FH_VENC_GetStream*`, `FH_VENC_ReleaseStream` | `MEDIA_STREAM_6`, ring-wrap decode and exactly-once `PAE_STREAM_STEP` release |
| IDR/runtime RC | `FH_VENC_RequestIDR`, `SetRCAttr`, `SetRcChangeParam` | native force-I operation plus the recovered stopped-channel/full RC and bounded realtime RC controls |
| MIPI/sensor | `mipi_init`, `API_ISP_SensorRegCb`, `SensorInit`, `SetSensorFmt` | GC1054/MIPI callback contract, board GPIO5 bootstrap and native sensor sequencing |
| ISP | `API_ISP_MemInit`, `Init`, `Run`, `Exit`, AE/AWB/mirror calls | recovered `/dev/isp` initialization/statistics/control path and platform ISP state machine |
| high-level image controls | `FHAdv_Isp_*` | adapter onto native ISP controls plus board-owned day/night/illumination policy |

The next adapter work should recover signatures/record layouts only for the smallest API slice being implemented. Cross-Fullhan names are semantic guidance; FH8852 structure layouts must never be copied into FH8626 code without evidence.

## Builder assembly

The Majestic device target is:

`fh8626v100_lite_anjia-ajl33pq0866_majestic`

Its Builder-local `.firmware` metadata binds it by default to the fork-local Firmware Majestic direction, so normal staging invocation is now simply:

```sh
bash builder.sh fh8626v100_lite_anjia-ajl33pq0866_majestic
```

Explicit `OPENIPC_FW_REPO` / `OPENIPC_FW_REV` values may still override that metadata for controlled bisect/debug work.

There is no separate active Majestic Builder branch and no copied Majestic board overlay. The composed FH8626 Builder targets remain CI-opted-out while their required Firmware state is fork-local.

No new Builder CI/build/hardware validation was run for the current `9c507b85481df2787bb214e535f17003bdebb6e6` Builder tip during the latest source-architecture/documentation pass.


## Evidence boundary

The **historical experiment** is hardware evidence for Majestic HTTP/control-plane viability on AJL33PQ0866.

The **new reconstructed branches** are source staging only. They have not yet been built or flashed, and must not inherit `HARDWARE_PASS` merely because they were reconstructed from the historical experiment.

The video/ISP path is explicitly unfinished. Do not enable media by default or call the Majestic direction complete until the SDK/sensor/ISP compatibility boundary is implemented and tested.

## Next gates

1. Owner-build Firmware `work/fh8626v100-majestic@c741f6f...` through Builder `work/fh8626v100-anjia` target `fh8626v100_lite_anjia-ajl33pq0866_majestic`; record resolved Buildroot config plus kernel/rootfs sizes.
2. Boot it and confirm Majestic process ownership, port 80, WebUI/haserl and board networking/services with media disabled.
3. Run `majestic-fh8626-abi-probe` and retain complete output. Also record `sha256sum /usr/libexec/majestic-fh8852v200/majestic` so this first reconstructed run is attributable to exact donor bytes.
4. Pin or otherwise make that donor Majestic binary reproducible; the current `master` S3 artifact is a moving input and no immutable donor object is currently indexed in `evidence/MANIFEST.tsv`.
5. Use the probe result and translation table to choose the smallest source adapter slice. Do not re-reverse already recovered FH8626 VMM/VPU/PAE/ISP operations.
6. First media acceptance remains VI -> VENC -> sustained RTSP. ISP tuning, JPEG, audio and board scene integration follow only after base H.264 is stable.
7. Do not copy old Firmware kernel patches, FH8626 factory blobs or FH8852 kernel-side payloads into this direction to make capture appear to work.


## Sensor ABI mismatch and first source adapter

The retained hardware experiment is now more precisely explained. The FH8852 Majestic build accepted the explicit native GC1054 path and logged `Using fh8626/libgc1054_mipi sensor` before its SDK path crashed, while direct bus-0 reads still returned GC1054 ID `0x10/0x54`. That result proved neither sensor failure nor simple loader failure.

A later static comparison closes the missing ABI question:

- native FH8626 GC1054 `Sensor_Create()` returns a 0x68-byte callback table;
- FH8852V200 GC4653, JXF32 and MN34425 plug-ins each return a 0x7c-byte table;
- their callback ordering differs materially, not merely by appended optional fields.

Firmware `work/fh8626v100-majestic@7fd1ee93...` therefore no longer feeds the native 0x68-byte object directly to an FH8852 consumer for the next media experiment. It builds an FH8852-shaped GC1054 facade and maps only recovered semantics onto the native FH8626 callback object. Proven mappings include VI attributes, initialization, format, register access, integration, gain and available AWB callbacks. FH8852-only or not-yet-proven operations are localized as staging stubs and can be forced to `-ENOSYS` with `FH8626_MAJESTIC_STRICT=1`.

This facade is transitional evidence tooling. It still expects the transitional native FH8626 GC1054 plug-in when the explicit media runner is used; it is not the final open sensor backend and it is not enabled by the default media-off service.

## Donor/native ioctl overlap

Static donor inspection also shows that not every FH8852 userspace layer needs replacement merely because the SoC differs. At least these literal operations overlap the recovered FH8626 contract exactly:

- FH8852 `libdsp.so` contains `MEDIA_BIND 0xC0084D00`;
- FH8852 `libdsp.so` contains `MEDIA_UNBIND_SRC 0xC0044D02`;
- FH8852 `libispcore.so` contains ISP start request `0x0000690A`.

This is evidence of partial ioctl-family continuity, not proof that the associated structures or complete libraries are binary-compatible. Adapter work should replace a donor layer only after its actual argument/layout contract is shown to differ.
