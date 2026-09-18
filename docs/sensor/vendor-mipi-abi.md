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

## Exact open-replacement contracts recovered 2026-09-18

Focused reverse in the canonical sensor project
`anjia_ajl33pq0866_fh8626v100_sensor_libs` confirmed enough of the active
GC1054 path to implement the normal 1280x720@25 sensor/MIPI bring-up without
guessing libc-private state.

The reusable source representation is
`source/fh8626v100/components/sensor/gc1054/gc1054_native_contract.h`.

### Sensor initialization

The FH8626 callback at table offset `+0x28` performs:

1. `mipi_init({5, 0, 0, 0, 0, 1})`;
2. `SensorDevice_Init(0x21, 0)`;
3. marks the sensor initialized;
4. seeds cached gain to `0x40`;
5. seeds cached integration time to `0xd0`.

ELF constructors for both sensor libraries contain only normal frame-info
registration. No hidden hardware initialization was found there.

### I2C wire boundary

For the active GC1054 mode, `SensorDevice_Init(0x21, 0)` opens
`/dev/i2c-0` and performs:

- ioctl `0x704` with argument `0`;
- ioctl `0x706` with argument `0x1a`;
- register transactions through ioctl `0x707`.

The `0x707` request is an array of 12-byte I2C message records. The message
address field is `0x21`. Mode `0` means one-byte register address plus
one-byte value for writes; reads use a one-byte register-address message
followed by a one-byte read message. Keep the per-message address `0x21`
distinct from the separate `0x706` argument `0x1a`; the stock object uses
both and their roles must not be collapsed by assumption.

### Complete sensor-format surface

Full-library reverse found five unique 145-write arrays. The first element of
every array is `{0xf2,0}`; there is **no sentinel**. GCC hoisted that first
element into local registers, which is why the decompiler initially made the
array pointer appear four bytes late.

| format | numeric alias | nominal fps | base frame length | VBLANK 0x07/0x08 | stock array |
|---|---:|---:|---:|---:|---|
| `0x80104e20` | — | 20.0 | 1125 | `0x0185` | `0x13448..0x1368b` |
| `0x8010411a` | — | 16.6666 | 1352 | `0x0268` | `0x1368c..0x138cf` |
| `0x80103a98` | — | 15.0 | 1500 | `0x02fc` | `0x138d0..0x13b13` |
| `0x80107530` | `4` | 30.0 | 749 | `0x000d` | `0x13b14..0x13d57` |
| `0x801061a8` | `3` | 25.0 | 899 | `0x00a3` | `0x13d58..0x13f9b` |

An inline Ghidra byte comparison proved that these arrays are identical except
pair 13/register `0x07` and pair 14/register `0x08`; 30 fps differs from
25 fps only at pair 14 because both high bytes are zero.

All five formats return the same 24-byte VI geometry except word16[0], the
base frame length. The remaining words are
`0x06be,720,1280,0,0,720,1280`; dword +0x10 is zero and dword +0x14 is zero
normally or `2` when orientation mode is non-zero.

Immediately after the final format array at `0x13f9c` begins the normal
Bayer-map `[0,3,1,2]`. Non-zero orientation mode uses
`[2,1,3,0]`.

### MIPI MMIO boundary

`libmipi.so::mipi_init` maps four physical regions through `/dev/mem`:

- `0xf0000000`, length `0x38`;
- `0xf0002000`, length `0x44`;
- `0xf1000000`, length `0x0c`;
- `0xf1100000`, length `0x14c`.

For the GC1054 argument `{5,0,0,0,0,1}` the exact observed programming
includes:

- `0xf0000000+0x0c |= 0x00010000`;
- clear `0x1c00` at `+0x24`;
- `+0x28 = (old & 0x003fff9f) | 0x60`;
- `0xf0002000+0x40 = 1`;
- initialization of the `0xf1100000` control block, including indirect data
  values `0x00010034`, `0x14`, `0x00010044`, and `5 << 1`, each
  committed through the control word at `+0x50`;
- a 100 us settle;
- the one-lane selector resulting from argument word 5 == 1 is `0`;
- `+0xe4/+0xf4/+0x104/+0x114/+0x124/+0x134 = 0xffffffff`;
- with the remaining GC1054 words zero, the `0xf1000000` mode word preserves
  only its existing `0x3f8` field and then writes `+0x04 = 1`.

These are exact active-path writes. Names for individual MIPI register fields
must remain conservative until independently identified.

### Exact gain/integration/frame-length programming

The active AE-facing callbacks are now source-reconstructable without the
vendor object.

Integration is direct: the requested value is cached and written on page 0 as
register `0x03 = integration >> 8` and `0x04 = integration & 0xff`.

Gain uses page 1 registers `0xb6/0xb1/0xb2` with exact thresholds recovered
from `Gc1054SetGain@0x118d4`, followed by page 4 register `0x40`:

- `0x40..0x5a` -> B6=0;
- `0x5b..0x7e` -> B6=1;
- `0x7f..0xb5` -> B6=2;
- `0xb6..0x100` -> B6=3;
- `0x101..0x170` -> B6=4;
- `0x171..0x202` -> B6=5;
- `0x203..0x2e0` -> B6=6;
- `0x2e1..0x406` -> B6=7;
- `0x407..0x5d2` -> B6=8;
- `0x5d3..0x823` -> B6=9;
- `>=0x824` -> B6=10 with both B1 and B2 scaled.

The exact integer arithmetic, including the compiler-equivalent high-word
multiplication constants, is preserved in
`fh8626_gc1054_gain_program()`. This avoids replacing target-proven rounding
with an approximate floating-point gain model.

After the page-1 triplet, page 4 register `0x40` is programmed from the
requested gain: 0 below `0x300`, 3 for `0x300..0x31f`, 4 for
`0x320..0x33f`, and 8 at `>=0x340`.

Frame-length programming is also closed for the active mode. Base frame length
is 899 lines. The `+0x14` callback multiplies that base by its integer input;
the helper then writes page-0 `0x07/0x08` with
`frame_length - 720 - 16`. The reusable contract exposes both calculations.

### Runtime sensor controls

The callback table now has source-level semantics for the active path:

- `+0x04`: gain programming;
- `+0x08`: VI attribute query;
- `+0x0c`: cached gain query;
- `+0x10`: integration programming through page-0 registers `0x03/0x04`;
- `+0x14`: frame-length/VTS update from the active format's base timing;
- `+0x18`: cached integration query;
- `+0x1c/+0x20`: mirror/flip set/query;
- `+0x28`: sensor/MIPI initialization;
- `+0x30`: I2C device close;
- `+0x34`: sensor format/register-table programming;
- `+0x3c`: direct register write;
- `+0x40`: maximum integration-delta query, returns `5`;
- `+0x4c`: named control/query surface;
- `+0x64`: command surface used for timing callbacks and orientation state.

The frame-length helper writes page-0 registers `0x07/0x08`. Gain and
integration setters are already fully visible in the retained Ghidra program.

### Teardown boundary

`Sensor_Destory` only clears the 0x68-byte callback object. The callback at
`+0x30` closes the sensor device fd. `libmipi.so` exports no corresponding
full MIPI deinit operation; its mapped regions are process-lifetime in the
stock object. This confirms that a clean replacement should own teardown
explicitly rather than imitating `dlclose` behavior.

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
