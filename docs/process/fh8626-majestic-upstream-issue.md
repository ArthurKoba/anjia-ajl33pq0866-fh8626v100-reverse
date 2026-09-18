# Draft: Majestic native support for Fullhan FH8626V100

Status: draft only. Repository owner must review and open the issue manually.

## Proposed title

Add native Fullhan FH8626V100 platform support

## Proposed issue body

We are porting OpenIPC to a Fullhan FH8626V100 camera and would like Majestic to support FH8626V100 as a first-class Fullhan platform rather than relying on an FH8852V200 userspace compatibility experiment.

Target:

- SoC: Fullhan FH8626V100;
- CPU: ARM1176JZF-S / ARMv6KZ, 32-bit ARM EABI soft-float;
- OpenIPC userspace: musl;
- kernel: Linux 4.9.129 Fullhan platform tree;
- sensor on the current board: dual GC1054 MIPI;
- native sensor/ISP mode exercised by the platform work: 1280x720 at 25 fps;
- downstream VPU/VENC scaling exists independently of the native sensor mode.

What is already demonstrated on hardware:

- an FH8852V200 Majestic ARM/musl build can execute on FH8626V100 when its required Fullhan userspace libraries are present;
- Majestic can reach normal startup and serve HTTP/WebUI on port 80 with media consumers disabled;
- the historical explicit GC1054 media attempt reached the SDK sensor path and then crashed, while direct I2C still returned GC1054 ID bytes 0x10/0x54;
- FH8626 native platform work independently has working/recovered contracts for GC1054 sequencing, ISP, VI/VPU, H.264 PAE/VENC, encoded-stream acquire/release, force-IDR, JPEG/MJPEG implementation work, lifecycle/teardown and RTX audio transport.

The FH8852V200 donor userspace closure observed for the control-plane experiment consists of:

- libadvapi.so
- libadvapi_isp.so
- libadvapi_smartir.so
- libdsp.so
- libisp.so
- libispcore.so
- libmipi.so
- libvmm.so

Static inspection of that baseline shows the Majestic-facing Fullhan surface includes the expected API families:

- VMM: FH_SYS_VmmAlloc/Free, Mmap/Munmap;
- system/media: FH_SYS_Init/Exit and VPU-to-encoder binding;
- VI/VPSS: system/channel memory, VI attributes, channel open/enable and frame control;
- VENC: create/configure/start/stop, GetStream/ReleaseStream, RequestIDR and rate control;
- sensor/MIPI: mipi_init and ISP sensor callback/init/format operations;
- ISP/image controls: API_ISP_* and FHAdv_Isp_*.

The FH8626 implementation does not require FH8852 kernel modules. We specifically want to keep the FH8626 kernel/media path native and source-maintainable.

Requested Majestic-side work:

1. add FH8626V100 as a supported Fullhan target/build;
2. expose or document the exact Fullhan platform ABI/types Majestic expects for SYS/VPSS/VENC/ISP/sensor/audio so an FH8626 backend can be implemented without guessing FH8852 structure layouts;
3. support the native FH8626 GC1054/ISP/VI/VENC path rather than requiring FH8852 kernel modules;
4. keep stream ownership compatible with acquire/release semantics and explicit IDR requests;
5. support clean startup, shutdown and same-boot restart;
6. once base video is established, allow the normal Majestic image/day-night controls and two-way audio integration to be wired to the FH8626 platform backend.

We can provide the recovered FH8626 ioctl/state-machine contracts and hardware-test results needed to implement and validate the backend. The current FH8852V200 compatibility package is intentionally treated as staging, not as a production architecture.
