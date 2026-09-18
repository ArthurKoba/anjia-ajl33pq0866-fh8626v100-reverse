# FH8626V100 proprietary blob retirement

Status: ACTIVE DEBT / PRESERVATION ONLY.

Checked: 2026-09-18.

This document tracks the remaining proprietary FH8626 media/runtime artifacts and the order in which they should be eliminated or given defensible vendor-SDK provenance. It is a roadmap, not a claim that every blob is already understood or replaceable.

The inventory comes from ArthurKoba/openipc-firmware/fh8626v100-platform@f4bf49da6ef355c9e733e00d774efe403513b1d4. Do not copy these binaries into this reverse repository. Heavy or unique binary evidence belongs in the external evidence store with hashes and provenance.

## Working rule

A working factory artifact may remain temporarily in the preservation branch while its role is being replaced. That does not make it acceptable final upstream content.

For every artifact establish: exact runtime role; callers and ABI; device/ioctl/MMIO contract; official SDK provenance if any; existing reconstructed-source coverage; owning repository; source verification; target hardware evidence. Remove the blob from the final shipped dependency set only after its contract is replaced.

Do not use LD_PRELOAD, kallsyms/private-offset hooks or runtime memory patching as a retirement strategy.

## Current preservation inventory

Kernel modules:
- bgm.ko
- enc.ko
- gpio_wave.ko
- isp.ko
- jpeg.ko
- media_process.ko
- vmm.ko
- xbus_rpc.ko

Userspace plug-ins:
- libmipi.so
- libgc1054_mipi.so

Firmware and sensor/profile objects:
- rtthread_arc.bin
- packaged gc1054_day.bin
- sensor_gc1054_mipi.bin
- source-tree gc1054_day.bin
- source-tree gc1054_night.bin
- source-tree gc1054_wlight.bin

## Exact inventory audit

Exact SHA-256 values, duplicate detection and ownership decisions for all 16 preserved binary paths are recorded in `docs/process/fh8626-firmware-ownership-audit.md`. There are 15 unique payloads because the packaged and source-tree `gc1054_day.bin` files are byte-identical.

The clean Firmware candidate `rework/fh8626v100-clean-integration@f9146dd42a2f606d305ebccd301268848de26880` ships none of these factory artifacts. This is architecture cleanup, not proof that every runtime contract has been replaced. The preserved WIP branch remains the recovery source until any still-needed unique payload without an external evidence locator has been externalized.

## Existing open coverage

The preserved source already contains substantial reconstructed ISP runtime/control, AE, AWB/CCM, GC1054 sensor-facing control, media ownership/lifecycle, geometry, dual-sensor sequencing, audio contracts and encoder/JPEG/BGM/media ABI contracts. The large userspace ISP/control problem is therefore substantially covered and should not be restarted wholesale.

This source coverage does not automatically replace isp.ko, vmm.ko, media_process.ko, enc.ko, jpeg.ko, bgm.ko, xbus_rpc.ko, gpio_wave.ko, the MIPI/sensor plug-ins or ARC firmware. Each remaining binary must be closed at its actual ABI/hardware boundary.

## Priority

### P1 — small userspace boundaries

1. libgc1054_mipi.so: map exports and callsites, compare against existing fh8626_sensor_gc1054.c, move sensor mode/register/exposure/gain control into source-built code, then hardware-test both GC1054 paths.
2. libmipi.so: map init/config/teardown ABI, bind it to the known MIPI/VI hardware contract, implement the minimal FH8626 source path, then validate cold start/restart and both sensors.

These are preferred before another broad ISP reverse pass.

### P2 — sensor/profile data

Classify sensor_gc1054_mipi.bin and day/night/wlight objects as executable code versus structured data. Where the format is understood data, keep reviewed source tables/structures and generate a runtime serialization if one is still required. Retain exact stock bytes externally as comparison evidence.

### P2 — separable kernel media modules

Work one subsystem at a time. Initial ordering:
1. gpio_wave.ko
2. bgm.ko
3. jpeg.ko
4. enc.ko
5. xbus_rpc.ko
6. media_process.ko
7. vmm.ko
8. isp.ko

This ordering is provisional. Reverse the dependency graph first; if a small-looking module is only a front-end to a larger private core, move it behind that core. isp.ko is late because it is the largest and most coupled kernel-side boundary while much userspace ISP behavior is already reconstructed.

### P3 — ARC/RTX firmware

Treat rtthread_arc.bin separately. Preferred outcomes: official redistributable Fullhan SDK firmware with exact provenance; SDK source and reproducible build; otherwise a documented isolated external firmware dependency while host-side audio remains open. Full clean-room replacement is a later task if actually required.

Do not block smaller .so or kernel-module cleanup on a complete ARC firmware rewrite.

## Definition of done

A blob is retired only when production no longer selects the opaque artifact; its replacement is source-built or has explicitly accepted SDK provenance; the consumed ABI/device behavior is documented; applicable source checks pass; relevant target behavior is hardware-tested; rollback/reference evidence remains retained externally; and the replacement lives in the correct owning repository.

Deleting a file without replacing its runtime contract is not retirement.
