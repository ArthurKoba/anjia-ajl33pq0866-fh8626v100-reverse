# FH8626V100 build, live-test and flash contract

This document records current operational rules for building and testing OpenIPC on the ANJIA AJL33PQ0866.

Use the component repositories and engineering refs described in `upstream-integration.md`, verifying their current remote state before each integration cycle.

## Build boundary

Build OpenIPC in a clean Linux environment. Keep Windows SDK/Git/Python/compiler paths out of the Buildroot toolchain environment.

The generic FH8626V100 platform configuration and the ANJIA product/device profile are distinct. A successful generic FH8626V100 build is not proof that the resulting image contains the camera-specific board/profile requirements.

The Builder device profile is the product-image assembly boundary. Verify the current Builder interface/profile before each integration cycle.

Kernel changes affect a product image only after they are propagated into the kernel source/patch input actually consumed by the build. Rebuild/invalidate the relevant package so stale Buildroot output cannot silently survive.

## Live candidate testing

The camera uses a read-only SquashFS lower layer with a persistent OverlayFS upper layer. Temporary candidates should run from a transient path such as `/tmp`, not overwrite packaged binaries under `/usr/bin`.

Before interpreting a live test, establish attribution:

1. candidate path/hash;
2. running PID;
3. `/proc/PID/exe` resolves to the candidate;
4. expected listener/resource belongs to that process;
5. the previous owner/process is no longer competing for the same resource;
6. the candidate reached its expected ready state.

A stale persistent overlay binary can mask a newly flashed SquashFS binary. Remove only the known stale override when necessary; do not wipe the entire overlay unless that is an intentional recovery action.

## Permanent changes

Permanent product changes belong in the repository that owns the component and must be rebuilt into the firmware/product image. Do not treat hand-copied binaries in the persistent overlay as the final integration mechanism.

## Upgrade boundary

For an already-running OpenIPC system, prefer the normal OpenIPC upgrade path so kernel and rootfs move together.

Do not rewrite bootstrap, U-Boot environment or U-Boot during an ordinary firmware update. Raw SPI/MTD writes belong to an explicit bring-up/recovery procedure.

After reboot, verify the actual running kernel/root layout and the processes relevant to the change. Successful flashing is not feature acceptance.

## Initial stock-to-OpenIPC bring-up

Initial migration from stock is a different procedure from an ordinary OpenIPC upgrade. Use the current U-Boot/board documentation and explicit readback/validation appropriate to the target.

A TFTP RAM boot is a test mechanism and does not itself modify flash. Keep temporary boot parameters transient unless a persistent change is intentional.

## Evidence boundary

Build or host-test success is not `HARDWARE_PASS`. New firmware candidates still require the target acceptance appropriate to the changed subsystem.
