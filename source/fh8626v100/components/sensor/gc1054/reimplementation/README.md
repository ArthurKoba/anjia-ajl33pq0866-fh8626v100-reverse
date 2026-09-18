# Source reimplementation of FH8626 sensor libraries

This directory is a source-level behavioral reconstruction of the retained
stock `libgc1054_mipi.so` and `libmipi.so`.

Evidence class: `REVERSE_CONFIRMED`.

The goal is to preserve the complete externally visible library behavior and
hardware transactions in auditable C, not to reproduce the original compiler
output byte-for-byte. ARM EABI compiler glue and soft-float helper routines are
represented by normal C semantics rather than copied disassembly.

## libmipi

`fh8626_libmipi_reimplementation.c` covers every non-ELF function and every
exported data symbol in the retained object:

- `FH_MIPI_Version` / `get_mipi_version`;
- `mipi_init`;
- `mipi_mm_init`, `mipi_mm_close`, `mipi_mmap`;
- exported `mipi_mm_fd`;
- exported 16-byte `mipi_common`.

The stock library has no MIPI deinit API. Four physical mappings are retained
for process lifetime after the first `mipi_init`; only the temporary
`/dev/mem` fd is reopened/closed around each map operation.

The generic six-word input semantics are retained exactly in the source. The
GC1054 library supplies `{5,0,0,0,0,1}`.

## libgc1054_mipi

`fh8626_libgc1054_mipi_reimplementation.c` covers the complete retained
GC1054 object's functional surface:

- every exported sensor/I2C/SPI/clock/environment helper;
- all 26 slots of the 0x68 `Sensor_Create` callback object;
- all five stock 1280x720 format arrays and both numeric aliases;
- cached gain/integration/frame-length state;
- exact gain, integration and VTS programming;
- mirror/flip register transforms and Bayer-format mapping;
- named `STD_FRAME_RATE`, `CUR_FRAME_RATE`, `REAL_FLIP_MIRROR` and
  `MAX_INTT_DIFF` queries;
- command IDs 1 and 0x80000..0x80003;
- all four I2C register/data-width modes and multi-message behavior;
- the external clock helper and the stock `SensorGetEnvInt` parsing quirk;
- stock teardown semantics: `Sensor_Destory` clears only the callback table.

The remaining compiler-generated routines in the original binary were ARM
soft-float/libgcc conversions, double multiply/divide/compare helpers, ELF
frame-registration glue and empty destructor helpers. They contain no
sensor-specific semantics and are represented by normal C arithmetic/runtime
behavior in this source reconstruction.

## Source↔Ghidra audit corrections

The post-reconstruction audit is performed against the migrated programs in
`anjia_ajl33pq0866_fh8626v100_apollo:/sensor/`.

The first audit pass corrected three subtle issues that were hidden by
decompiler presentation rather than missing binary coverage:

- callback slots `+0x1c/+0x20` tail-call the low-level mirror/flip helpers and
  therefore preserve their integer return value; the source callbacks return
  `int`, not `void`;
- command `0x80002` uses a signed target-fps callback output;
- command `0x80002` still invokes the registered adjustment callback for an
  unsupported format, passing nominal fps `-10000` with base frame length
  zero, instead of returning before the callback;
- raw `I2CSensor_Read` uses 2-byte reads for every mode other than 0 and 2,
  including mode values >=4; this is not equivalent to a parity-only rule;
- the VI-attribute builder zeroes all 24 output bytes before format dispatch,
  so an unsupported format returns failure with a zeroed output structure.

The earlier mistaken interpretation of a trailing `0/0` format-table
sentinel was also removed: all five arrays are exactly 145 pairs and the data
following the 25-fps array is the Bayer map.

## Boundary

This is complete static/reverse functional coverage of the two retained
userspace sensor libraries, not yet hardware acceptance of rebuilt shared
objects. The source is intended as the auditable replacement specification;
target compile/link/runtime validation remains a separate evidence gate.
