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

### Active sensor format

Format `0x801061a8` and legacy numeric alias `3` select the same active
1280x720@25 register script.

`Gc1054SetFormat` first writes register `0xf2=0`, then walks the exact
uint16 register/value table at stock address `0x13d5c..0x13fa0`. The final
`0/0` pair is a sentinel and is not executed. The promoted source array
contains 145 executed writes: the separate leading `0xf2=0` plus 144 table
entries.

The corresponding raw 24-byte VI attribute result is:

- eight 16-bit words:
  `899, 0x06be, 720, 1280, 0, 0, 720, 1280`;
- 32-bit word at +0x10: `0`;
- 32-bit word at +0x14: `0` normally, `2` when the plugin orientation
  state is non-zero.

Field naming remains tied to the ISP consumer contract; the raw values above
are exact and should be preserved even where semantic names remain under
analysis.

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
