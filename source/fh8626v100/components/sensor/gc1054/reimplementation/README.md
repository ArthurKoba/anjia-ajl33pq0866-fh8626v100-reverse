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

## Boundary

This reference implementation is not yet hardware acceptance of a rebuilt
shared object. Hardware validation remains separate from static/reverse
completeness.
