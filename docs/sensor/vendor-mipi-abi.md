# Vendor sensor / MIPI ABI boundary

This document records the compatibility boundary for the recovered FH8626V100 GC1054 sensor and MIPI vendor objects. It is a porting contract, not a recommendation to ship proprietary userland binaries.

## Exact target ABI

Stock Apollo and the exercised GC1054/MIPI objects are ARM32 little-endian EABI5 soft-float objects built for the uClibc userspace environment.

Recovered dependencies show a narrow interface class:

- `libgc1054_mipi.so` depends on libc plus `mipi_init` and ordinary wrappers such as `ioctl`, `open/close`, `usleep`, `malloc/free`, `getenv`, `strtol`, `memset/strcmp`, and simple formatting/output;
- `libmipi.so` uses `mmap/munmap`, `open/close`, `usleep` and simple formatting/error helpers.

The native FH8626 GC1054 `Sensor_Create()` contract is now explicitly sized: it returns a **0x68-byte callback table**. Recovered offsets include name at +0x00, gain at +0x04, VI attributes at +0x08, integration at +0x10, initialization at +0x28, format at +0x34, register write at +0x3c and the common control/query surface at +0x4c.

Static inspection of three independent FH8852V200 sensor plug-ins (`gc4653_mipi`, `jxf32_mipi`, `mn34425_mipi`) shows a different, internally consistent **0x7c-byte callback table**. Representative FH8852 offsets are: GetSensorViAttr +0x04, Sensor_Init +0x14, Sensor_DeInit +0x1c, SetSensorFmt +0x20, Sensor_Kick +0x24, SetSensorReg +0x28, GetSensorReg +0x3c, GetAEDefault/GetAEInfo +0x50/+0x54, SetIntt +0x58, SetGain +0x60 and Sensor_Isconnect +0x78.

Therefore a native FH8626 `libgc1054_mipi.so` callback object is **not ABI-compatible** with the FH8852V200 ISP merely because both expose `Sensor_Create`. Passing the 0x68-byte object directly to an FH8852 consumer crosses a proven structure-layout mismatch.

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
