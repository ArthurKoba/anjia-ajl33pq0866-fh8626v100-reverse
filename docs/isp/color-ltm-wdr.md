# ISP color / CCM / LTM / WDR findings

## Color/CCM historical reverse

Focused reverse recovered exact arithmetic/table behavior for several stock ISP routines, including D0238/CFEB0, D1258, D1724 and D1DB0/D14 coefficient paths. Some earlier approximate values and isolated register replay interpretations were later corrected by exact reconstruction.

An isolated write of recovered D0238 output words did not remove a visual artifact. This did not invalidate the reverse: later exact reconstruction reproduced all nine stock words bit-for-bit, showing the subroutine depends on coherent pipeline context rather than being a standalone visual fix.

## Historical AWB -> CCM diagnostic step

A v4.2.1 source-only diagnostic patch attempted a coherent downstream AWB/CCM transaction rather than treating CCM words as isolated writes. For operator-supplied gains it derived the persistent ratio fields at `ctx+0xA8/+0xAA`, ran the reconstructed C9F68 selection/interpolation path, applied the CE670 interpolation/CE628 six-word packing model, and wrote the resulting six words to ISP `+0x4C0..+0x4D4` while temporarily freezing the writer-facing ISP enable. The package explicitly said it did not implement CAFC0 and did not replace the already-proven `awbmode1` step.

This was an implementation/diagnostic candidate, not a target or hardware acceptance result. Its useful lesson is the dependency boundary: AWB-derived persistent state, CCM selection/interpolation and the six packed MMIO words must be treated as one coherent state transition; isolated final-register replay is not equivalent.

## Dynamic LTM execution parity

A focused historical Ghidra pass went beyond decompiler matching and executed the relevant Apollo ARM instructions under a bounded emulator.

`REVERSE_CONFIRMED` within the tested domain:

- the LTM polynomial path was executed for 3376 vectors / 3,962,329 ARM instructions; production C matched all 3360 cases inside the public `>=64`-pixel domain;
- the full-state path was executed for 32 fixtures / 446,912 ARM instructions; all 24 history words and six output-register words matched the replacement C;
- `D0528` publishes record-2 min/max on every invocation, including the first outer `D0DF4` call;
- `D0630` samples source1/bias/old-`0x24C` before its internal `D0528` scan, rereads geometry afterward, shifts history/calculates the target before source0, and consumes source2 after source0 normalization;
- register `0x248` receives two ordered volatile RMW operations;
- `D0B2C` mode 1 publishes the full `0x00ffffff` word at `0x254`; retaining the previous high byte was a replacement bug;
- `D0B2C` code ends before the following literal-data entries; those literals are not function instructions.

The emulator used synthetic RAM/GOT/context/MMIO fixtures and strict unknown-read faults. This proves bounded numerical/state equivalence, not physical DMA-epoch timing, exhaustive floating-point equivalence or visual acceptance. The historical replacement candidate carrying these fixes was source-only and not deployed in that batch.

For future native-owner work, preserve volatile sampling order and full-word/masked publication semantics; do not rewrite the proved polynomial merely to chase image appearance.

## Public LTM attribute ABI

The same reverse work recovered the stock public LTM attribute surface around `0x2163FC/0x216938`:

- the public record is 80 bytes;
- Set performs sequential in-place clamps and packs twelve nibble fields into the context representation;
- a null attribute returns stock error `-3002`;
- Get leaves byte `0x4F` untouched;
- public Get/Set do not themselves publish MMIO or set a dirty/update flag;
- helper `0x78900` delegates to `0xDF1E0`; it is not a second independent LTM implementation.

The historical owner exposed separate “hardware enable” and “periodic updates” controls. Immediate MMIO application in that owner was explicit replacement policy so that an off request could work while periodic updates were paused; it must not be attributed to the stock public Set routine.

## Shared LUT / exact consumer status

The durable shared `0x316CA8` Q7-like sine-LUT conclusion applies to D1258, CFEB0/D0238 and the LUT-dependent part of D1DB0. Working instruction-level detail and annotations belong in the canonical Ghidra MCP project. Current Git keeps the reusable conclusion and implementation boundary rather than depending on historical report filenames or a Git-side reverse export tree.

## LTM/WDR boundary

WDR/LTM remains a focused open boundary only if native ISP work becomes necessary. Do not reopen broad ISP reverse preemptively.

## Evidence rule

Current implementation claims should depend on durable Git contracts, canonical Ghidra MCP state, or retrievable external objects indexed in `evidence/MANIFEST.tsv`. Historical report names and generated subsets are provenance, not a second current authority.
