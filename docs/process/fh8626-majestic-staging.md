# FH8626V100 Majestic staging

Status: `OFFLINE_PORT_COMPLETE / HISTORICAL_CONTROL_PLANE_HARDWARE_PASS / NEW_BUILD_AND_HARDWARE_PENDING`.

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
- builds a source `majestic-fh8626/libgc1054_mipi.so` facade for the proven FH8852 0x7c -> FH8626 0x68 sensor mismatch;
- builds source `libvmm.so` and `libdsp.so` facades for the recovered FH8626 VMM/SYS/VPSS/VENC boundary;
- translates native FH8626 stream descriptors into the FH8852 public stream representation and preserves exactly-once descriptor release;
- includes strict, permissive-stub and fixed native-video runners; native-video is constrained to the recovered 1280x720@25 H.264 Baseline/VBR contract;
- keeps the normal init service media-off;
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

1. Owner-build Firmware `work/fh8626v100-majestic@718f6a14...` through Builder `work/fh8626v100-anjia` target `fh8626v100_lite_anjia-ajl33pq0866_majestic`; record resolved Buildroot config plus kernel/rootfs sizes.
2. Boot the default media-off service and confirm Majestic directly owns port 80 with stock WebUI/API/WebSocket behavior. Characterize `/metrics` as served by Majestic itself; do not insert a proxy or frontend patch.
3. Run `majestic-fh8626-abi-probe` and retain complete output. Also record `sha256sum /usr/libexec/majestic-fh8852v200/majestic` so the run is attributable to exact donor bytes.
4. Run strict media mode first and preserve the first failing API/callsite. Run permissive-stub only to expose later optional call ordering. Then run native-video mode.
5. Native-video acceptance is VI -> VENC -> balanced descriptor acquire/release -> sustained RTSP at the fixed 1280x720@25 H.264 Baseline/VBR contract.
6. After base H.264 is stable, validate the staged RTX audio path and characterize JPEG/ISP behavior. The donor JPEG kernel request family and substantial donor ispcore request subset already match FH8626 literally, so preserve compatible donor code unless a concrete public-record/layout mismatch is observed; do not replace layers merely because the SoC name differs.
7. Regress shutdown/restart, PTZ, lens, illumination and storage.
8. Pin/reproduce the exact donor binary before product acceptance. Do not copy old Firmware kernel patches, factory blobs or FH8852 kernel-side payloads into the design.


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

- FH8852 `libdsp.so` contains `MEDIA_BIND 0xC0084D00` and `MEDIA_UNBIND_SRC 0xC0044D02`;
- the donor JPEG core uses the same recovered FH8626 requests for memory query/init/uninit, channel config, MJPEG config, start, stop and release: `0xC0104A02`, `0xC0184A00`, `0xC0184A01`, `0xC0104A03`, `0xC0344A05`, `0xC0044A09`, `0xC0044A0A`, `0xC0044A10`;
- donor `libispcore.so` contains exact FH8626 requests `0x6919`, `0x40016920`, `0x40016921`, `0x690A`, `0x40046924`, `0x40016911`, `0x40046930`, `0x8010690E` and `0x40046908`;
- donor `libisp.so` and donor `libvmm.so` do not show the same direct-literal overlap for the recovered native request set; VMM is therefore explicitly replaced in the source-first media path.

This is evidence of partial ioctl-family continuity, not proof that the associated structures or complete libraries are binary-compatible. Adapter work should replace a donor layer only after its actual argument/layout contract is shown to differ.


## HTTP/WebUI and metrics boundary

Majestic continues to own its normal HTTP port, API, WebSocket and frontend directly. No JavaScript rewrite and no auxiliary HTTP proxy are part of the accepted staging architecture.

The historical empty `GET /metrics` result is therefore still a real backend/provider blocker. CPU, RAM, uptime and network source data are available from Linux, but any fix must land in the actual Majestic/platform metrics-provider boundary (or official Majestic FH8626 support), not by hiding the route behind a second web server.


## RTX audio compatibility facade

Static reverse of FH8852 `libacw_mpi.so` showed that its public MPI is a thin wrapper over the same RTX command family already hardware-proven on FH8626. Matching contracts include reset `0x40000000`, command transport `0x20000000`, init `0x01040004`, config `0x01040005`, AI frame `0x01008000`, AO frame `0x01008002`, and the same simple command IDs for AI/AO enable/disable, volume, mode, clear and playback completion.

The recovered donor shared-memory layout also proves that the AO staging region is the tail of the RTX mapping: `mapped_base + (map_length - tail_length)`, with the corresponding transport offset derived from `map_offset`. Firmware therefore now builds a source `libacw_mpi.so` facade with AI frame/PTS retrieval and AO frame submission over native `/dev/rtxbus`.

Advanced AEC/AGC/NR semantics remain explicit unsupported boundaries until a concrete Majestic call requires their exact records. Physical speaker amplifier policy remains board-owned and is not embedded into the generic compatibility library.


## Final offline Ghidra audit

The final offline pass used the canonical ANJIA/FH8626 Ghidra projects rather than treating Divinus as authority:

- `anjia_ajl33pq0866_fh8626v100_apollo`;
- `anjia_ajl33pq0866_fh8626v100_isp`;
- `anjia_ajl33pq0866_fh8626v100_enc`;
- `anjia_ajl33pq0866_fh8626v100_media_process`;
- sensor/MIPI projects plus existing VMM/JPEG/kernel/ARC projects as needed.

Concrete results applied to Firmware:
- `FH_VPSS_Enable(channel)` passes the channel id to `0xC004694D`; it is not a boolean enable flag;
- VPU disable is the distinct no-payload `0xC004694E` request;
- `FH_VPSS_CloseChn(channel)` is `0xC0046950`;
- frame control is exactly two words: channel plus packed low16/high16 ratio;
- `FH_VENC_CreateChn` consumes `{support_type, capacity_width, capacity_height}`;
- normal/smart H.264 support bits are 0x4/0x8;
- the kernel PAE config remains exactly 0x2c bytes and full RC exactly 0x54 bytes;
- stock Apollo contains a combined H.264 public-attribute translator that programs PAE config first and full RC second;
- stock VENC order is `VPSS attr/open/framectrl -> VENC create/attr/start -> SYS bind`, while VI/VPU enable is owned separately;
- stock ISP image APIs mutate a shared userspace context and are not simple direct-ioctl wrappers.

Because the ISP controls depend on a complete shared userspace context, donor `libisp.so`/`libispcore.so` are deliberately retained as isolated transitional components until the target run proves a concrete mismatch. Replacing them with a partial facade before target evidence would be less correct, not more open.

All Divinus-specific mismatches found during this clean-room audit are tracked separately in `docs/process/fh8626-divinus-correction-ledger.md`.
