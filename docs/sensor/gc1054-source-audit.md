# GC1054 / MIPI source-to-Ghidra audit

Status: `REVERSE_CONFIRMED / SOURCE_RECONSTRUCTION_AUDITED`.

Canonical working programs:

- Ghidra project: `anjia_ajl33pq0866_fh8626v100_apollo`
- `/sensor/libgc1054_mipi.so`
- `/sensor/libmipi.so`

The temporary standalone `sensor_libs.gpr` project was removed after its
programs and analysis state were migrated into Apollo.

## Coverage

The audit accounts for every non-thunk functional routine in both retained
objects. The only original routines not reproduced as named sensor/MIPI C
functions are normal compiler/runtime mechanics:

- ELF init/fini and frame-info registration;
- empty compiler-generated destructor helpers;
- ARM soft-float/libgcc unsigned/signed integer-to-double conversion;
- double multiply/divide and exceptional-value helpers;
- double comparison wrappers;
- double-to-signed/unsigned integer conversion.

The reconstructed C uses the corresponding language/runtime arithmetic instead
of copying compiler-generated assembly.

## Export surface

The GC1054 reconstruction preserves the functional exports:

`set_clk_rate`, SPI stubs, all I2C helpers, all Sensor read/write helpers,
`SensorDevice_Init`, `SensorDevice_Close`, `SensorGetEnvInt`,
`GetDefaultParam`, `GetContrast`, `GetSaturation`, `GetSharpness`,
`GetMirrorFlipBayerFormat`, `GetSensorAwbGain`, `GetSensorLtmCurve`,
`Sensor_Create` and `Sensor_Destory`.

The MIPI reconstruction preserves:

`FH_MIPI_Version`, `get_mipi_version`, `mipi_init`, `mipi_mm_init`,
`mipi_mm_close`, `mipi_mmap`, exported `mipi_mm_fd`, and exported
`mipi_common`.

Linker-generated ELF entry/init/fini symbols are intentionally left to the
toolchain.

## Callback table

`Sensor_Create` zeroes exactly 0x68 bytes and populates these non-null slots:

| offset | behavior |
|---:|---|
| +0x00 | `"gc1054_mipi"` |
| +0x04 | set gain |
| +0x08 | get VI attributes |
| +0x0c | get cached gain |
| +0x10 | set integration |
| +0x14 | set VTS multiplier |
| +0x18 | get cached integration |
| +0x1c | set mirror/flip, returns low-level rc |
| +0x20 | get mirror/flip, returns low-level rc |
| +0x28 | sensor/MIPI initialization |
| +0x2c | ARM passthrough callback (`bx lr`) |
| +0x30 | sensor device close |
| +0x34 | set format |
| +0x3c | direct register write wrapper |
| +0x40 | max integration delta = 5 |
| +0x4c | named control/query |
| +0x64 | command interface |

All other slots remain zero.

## Data and format boundary

The five physical format arrays are exactly 0x244 bytes / 145 pairs each.
There is no sentinel. A Ghidra byte-comparison script proved they differ only
at the `0x07/0x08` VBLANK pair.

The initialized GC1054 data state is also accounted for: fd `-1`, cached gain
`0x40`, cached integration `0x6f0`, and the four-word oriented Bayer cache
initialized to `-1`. The rest of the sensor state is zero-initialized BSS.

## Remaining gate

This audit establishes source/static equivalence intent, not target runtime
equivalence. A rebuilt ARM1176 soft-EABI object still requires compile/link
inspection and physical-camera validation before the proprietary objects can be
declared retired on hardware.
