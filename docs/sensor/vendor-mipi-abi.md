# Vendor sensor / MIPI ABI boundary

This document records the compatibility boundary for the recovered FH8626V100 GC1054 sensor and MIPI vendor objects. It is a porting contract, not a recommendation to ship proprietary userland binaries.

## Exact target ABI

Stock Apollo and the exercised GC1054/MIPI objects are ARM32 little-endian EABI5 soft-float objects built for the uClibc userspace environment.

Recovered dependencies show a narrow interface class:

- `libgc1054_mipi.so` depends on libc plus `mipi_init` and ordinary wrappers such as `ioctl`, `open/close`, `usleep`, `malloc/free`, `getenv`, `strtol`, `memset/strcmp`, and simple formatting/output;
- `libmipi.so` uses `mmap/munmap`, `open/close`, `usleep` and simple formatting/error helpers.

No recovered GC1054/MIPI boundary required complex libc-private structures such as pthread internals, `stat`, `timespec` or another opaque vendor-owned object.

## Porting decision

- GC1054 sensor callback boundary: shim-feasible, but a clean/open implementation from the recovered callback/register contract is preferred.
- MIPI boundary: shim-feasible, but an open typed implementation around the recovered device/MMIO/ioctl contract is preferred.
- Full Apollo: if ever reused for focused engineering, isolate it as a uClibc helper process; do not link the full application into a musl-native OpenIPC process.
- ISP/media HAL: prefer an open implementation using recovered ioctls/structures/state contracts rather than opaque in-process binary reuse.

## Critical compatibility rule

ARM EABI5 soft-float compatibility does **not** imply uClibc and musl structure compatibility.

Any future boundary that crosses `stat`, `off_t`, `time_t`, `timespec`, pthread internals or another libc-owned layout must be independently typed/probed or isolated across a process boundary.

The narrow GC1054/MIPI callback/ioctl surface avoids most of those high-risk ABI structures, which is why it is a viable transitional engineering boundary.

## `mmap` compatibility caution

Historical musl owner/probe work required a compatibility wrapper because vendor uClibc-era code expects the older byte-offset mmap convention while Linux ARM `mmap2` uses page offsets. Treat such wrappers as compatibility scaffolding, not part of the clean final sensor API.

## Evidence boundary

This is `REVERSE_CONFIRMED` implementation-facing ABI classification. Target hardware acceptance of a particular replacement implementation remains a separate gate.

Further ABI reverse should use the canonical Ghidra MCP project and only when a concrete integration question requires it.
