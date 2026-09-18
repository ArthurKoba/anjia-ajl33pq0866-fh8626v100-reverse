# FH8626V100 Majestic staging

Status: `STAGED / HISTORICAL_CONTROL_PLANE_HARDWARE_PASS / NEW_BRANCH_BUILD_PENDING`.

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
- Divinus direction: `work/fh8626v100-divinus@0b12c87c202b12733b0a1b535b56d66891e4ca93`;
- Majestic direction: `work/fh8626v100-majestic@7ed2a17a67fce0c2ee8d80bc798cafe81cfa6c38`.

Builder:

- ANJIA/Divinus development line: `ArthurKoba/openipc-builder/work/fh8626v100-anjia@dac8d565aaa493c4fd83334df3054138b92ed01a`;
- ANJIA Majestic staging: `work/fh8626v100-anjia-majestic@85c496eab79e662cc2e0e511540265b969ae227a`.

Core platform/kernel fixes should be made on the shared core and then reconciled into both runtime directions. Majestic-specific compatibility code must stay on the Majestic direction unless it becomes demonstrably shared.

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
- installs **no** FH8852 kernel modules, firmware, load scripts or sensor plug-ins.

This is intentionally a compatibility staging package, not a statement that FH8852 userspace blobs are the final FH8626 media architecture.

## Builder assembly

The Majestic device target is:

`fh8626v100_lite_anjia-ajl33pq0866_majestic`

Builder can consume the fork-local Firmware branch without copying its package back into Builder:

```sh
OPENIPC_FW_REPO=https://github.com/ArthurKoba/openipc-firmware.git
OPENIPC_FW_REV=work/fh8626v100-majestic
bash builder.sh fh8626v100_lite_anjia-ajl33pq0866_majestic
```

Both FH8626 Builder targets remain CI-opted-out until their required Firmware state is available to the normal upstream clone path.

## Evidence boundary

The **historical experiment** is hardware evidence for Majestic HTTP/control-plane viability on AJL33PQ0866.

The **new reconstructed branches** are source staging only. They have not yet been built or flashed, and must not inherit `HARDWARE_PASS` merely because they were reconstructed from the historical experiment.

The video/ISP path is explicitly unfinished. Do not enable media by default or call the Majestic direction complete until the SDK/sensor/ISP compatibility boundary is implemented and tested.

## Next gates

1. Owner-build the exact Firmware/Builder Majestic branch pair and record resolved Buildroot config plus kernel/rootfs sizes.
2. Boot it and confirm Majestic process ownership, port 80, WebUI/haserl and board networking/services with media disabled.
3. Pin or otherwise make the donor Majestic binary reproducible; the current `master` S3 artifact is a moving input.
4. Only then resume media compatibility work. First target VI/VENC/sustained RTSP; ISP tuning and audio follow later.
5. Do not copy old Firmware kernel patches, FH8626 factory blobs or FH8852 kernel-side payloads into this direction to make capture appear to work.
