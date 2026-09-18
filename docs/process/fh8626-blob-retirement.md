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

The clean Firmware candidate `rework/fh8626v100-clean-integration@c437d6eb62ade81595c20cbb765b8ad10300e3e7` ships none of these factory artifacts. This is architecture cleanup, not proof that every runtime contract has been replaced. The preserved WIP branch remains the recovery source until any still-needed unique payload without an external evidence locator has been externalized.

## Mandatory reverse/recovery backlog

Every opaque payload in the preservation inventory is an active **source-recovery or reverse/reimplementation** task until production no longer depends on factory-extracted bytes.

For each payload use this order:

1. search Fullhan SDK releases, source drops, vendor mirrors and related SoC SDKs for the exact or compatible source implementation;
2. if buildable source is found, establish version/ABI compatibility, integrate a reproducible source build and validate it on the target; detailed binary reverse is then optional except where needed to prove compatibility or fill missing source;
3. if suitable source is not found or is incomplete, reverse the factory binary far enough to recover its ABI/data/protocol/hardware contract and implement the missing functionality from maintainable source;
4. keep the original extracted bytes only as evidence/reference. They are not an acceptable final production dependency.

A runtime `.ko`, `.so` or `.bin` may still exist as a **build output generated from retained source**. What must disappear is the checked-in or externally copied opaque factory payload.

When reverse/reimplementation is required, the minimum result for each class is:

- kernel module: recover module dependencies, exported/imported symbols, init/exit flow, device nodes, ioctl/proc/sysfs ABI, shared-memory/RPC contracts, relevant MMIO/IRQ/DMA behavior and caller ordering; then implement or recover a source-built replacement;
- userspace plug-in: recover exports, callsites, data structures, initialization/teardown, sensor/MIPI register or ioctl transactions and error semantics; then replace the opaque plug-in with source-owned implementation;
- sensor/profile `.bin`: determine whether it is code or structured data, recover the exact file/record format and field semantics, decode the stock object into reviewed source data, and regenerate equivalent runtime data when a serialized object is still required;
- ARC firmware: recover the host-to-ARC loading path, mailbox/RPC protocol, command and data structures, buffer ownership, audio timing/format contracts and failure/reset behavior. If reproducible Fullhan source is later found, use it as the implementation source; otherwise the recovered protocol remains the basis for an open replacement. An opaque SDK copy alone is not completion.

Per-payload required work:

| Payload | Mandatory reverse/recovery result |
| --- | --- |
| `bgm.ko` | Recover BGM device/ABI, shared buffers and media-pipeline dependencies; replace with source-built kernel/userspace boundary as appropriate. |
| `enc.ko` | Recover encoder control/stream ABI, buffer lifecycle, IRQ/DMA interactions and dependencies; replace with source-built implementation. |
| `gpio_wave.ko` | Recover waveform device/ioctl contract, timer/PWM/GPIO behavior and callers; replace with source-built implementation. |
| `isp.ko` | Recover ISP kernel ABI, register/memory contracts, statistics/control paths and dependencies needed by the open ISP runtime; replace the opaque kernel boundary. |
| `jpeg.ko` | Recover JPEG configuration, buffer ownership, completion/error ABI and dependencies; replace with source-built implementation. |
| `media_process.ko` | Recover media graph/process-control ABI, shared-memory/buffer contracts and lifecycle; replace with source-built implementation. |
| `vmm.ko` | Recover vendor memory manager allocation/mapping/cache ABI, address-space rules and consumers; replace with an open memory-management boundary. |
| `xbus_rpc.ko` | Recover RPC transport framing, endpoints, shared-memory/mailbox behavior and users; replace with source-built transport. |
| `libmipi.so` | Recover complete MIPI init/config/start/stop ABI and hardware transaction sequence; move it into source-owned FH8626 HAL code. |
| `libgc1054_mipi.so` | Recover GC1054 mode/register/exposure/gain ABI and exact call contract; move it into the source GC1054 implementation. |
| `rtthread_arc.bin` | First search for Fullhan ARC/RT-Thread source/build inputs. If complete source exists, build the firmware reproducibly from it; otherwise recover loader plus host/ARC RPC and audio-service protocol and implement the missing firmware/service from source. The factory-extracted image remains evidence only. |
| `sensor_gc1054_mipi.bin` | Identify executable/data format, decode all fields/records and recover the sensor contract into source; regenerate only if runtime serialization remains required. |
| `gc1054_day.bin` | Decode profile format and semantics into reviewed source data; regenerate deterministically if still required. |
| `gc1054_night.bin` | Decode profile format and semantics into reviewed source data; regenerate deterministically if still required. |
| `gc1054_wlight.bin` | Decode profile format and semantics into reviewed source data; regenerate deterministically if still required. |

The packaged and source-tree `gc1054_day.bin` paths are byte-identical, so they are one reverse task even though both preservation paths remain recorded.

Reverse work should use the canonical Ghidra MCP project and existing recovered source/contracts first. Do not redo already established userspace ISP/AE/AWB/media logic merely because a kernel blob remains opaque; target the unresolved ABI/hardware boundary of that blob.

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

Treat rtthread_arc.bin separately. First search for reproducible Fullhan SDK source/build inputs. If they exist and reproduce the required firmware/API, use that source and document the host/ARC interface needed for integration; a full binary reverse is unnecessary. If source is absent or incomplete, recover the loader/RPC/mailbox/audio contracts and implement the missing pieces. The factory-extracted ARC image, or an identical opaque SDK copy, may remain only as comparison evidence and must not be the final production artifact.

Do not block smaller .so or kernel-module cleanup on a complete ARC firmware rewrite.

## Definition of done

A blob is retired only when production no longer selects factory-extracted opaque bytes; its replacement is reproducibly built from retained vendor/open source or from a source implementation reconstructed from reverse engineering; the required ABI/device behavior is documented; applicable source checks pass; relevant target behavior is hardware-tested; rollback/reference evidence remains retained externally; and the replacement lives in the correct owning repository.

Deleting a file without replacing its runtime contract is not retirement.
