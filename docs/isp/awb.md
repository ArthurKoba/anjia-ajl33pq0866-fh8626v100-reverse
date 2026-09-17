# ISP AWB findings

Confirmed source-level mismatch in frozen native Divinus:

Exact owner behavior selects the runtime statistics bank/root, reattaches/rebinds the selected runtime root, then consumes statistics at `selected_runtime + 0x48`.

Frozen native candidate instead reads a fixed location equivalent to `isp_cfg + 0x1487b8`.

The earlier historical assumption that this fixed root was always authoritative is superseded. This mismatch must be repaired or deliberately justified before any clean native target acceptance run.

Do not infer hardware impact solely from the source mismatch. Hardware verdict remains separate.

## CA13C spatial-cell contract

A 2026-09-04 reconciliation of the stock Apollo path recovered a second durable AWB boundary. `CA13C` iterates **nine** 16-byte AWB statistic cells at module offsets `+0x10` through `+0x90`.

A historical replacement-owner implementation passed nine cells to the routine but accumulated only eight. That source mismatch was corrected and a focused ninth-cell-only regression passed. The omission is a credible mechanism for spatially dependent white-balance error when a strong white region occupies the ignored cell, but it is not by itself proof that every observed green cast came from this bug.

Treat the nine-cell iteration count and layout as `REVERSE_CONFIRMED`; treat the historical replacement-source defect/fix as `SOURCE_CONFIRMED`. Any present Majestic or resumed Divinus/native implementation must be checked independently rather than assuming the old fix is still present.

## Dispatcher, fallback and recovery semantics

Focused current-Ghidra reconciliation on 2026-09-05 recovered several exact control-flow details that matter if the native owner is resumed:

- `CA4F4` fallback returns logical gains without publishing `ISP+0x4BC` or `ctx+0xAC`; a replacement must not turn fallback into an unconditional brightness-correction write;
- the recovery counter advances only after viable statistics; during active recovery (`<=20`) stock uses gate fallback or aggregate-only gains, and the 21st viable attempt returns to the robust estimator;
- repeated invalid grids therefore do not consume the recovery window;
- `CA428` mode 0 publishes the literal default measurement words; mode 1 publishes context measurement parameters at the stock inactive/fallback slots;
- `CA13C` mode-0 normalization uses mask `0x1FFE00`;
- the estimator tails share unsigned-divisor, signed-16 correction and low-32-bit multiply/shift semantics;
- the shared dispatcher captures mode/low-9 configuration before the statistics query and owns the mode 0/1 versus mode 2/3 split.

These are `REVERSE_CONFIRMED` stock semantics. Historical replacement fixes and host tests are `SOURCE_CONFIRMED`; they are not current Majestic acceptance.

## CAFC0 and normalization arithmetic

The preceding 2026-09-05 control/arithmetic audit also recovered a bounded stock contract around `CAFC0`, `CB580/CB7B0`, `D27C4`, `C9F30`, `D282C` and `C9F68`:

- CAFC0 uses hold/dwell thresholds, signed-16 wrapped steps, direct/smoothed gains, BLC scaling and persistent sensor-gain split;
- optional callback order matters: the callback may mutate flags that are then reloaded before final low-word OR packing;
- CB580/CB7B0 normalization uses persistent gains, optional provider query and a low-32-bit quotient after a 64-bit numerator;
- D27C4 zero arithmetic is unsigned; C9F30 square/dot arithmetic preserves ARM low-32-bit behavior;
- D282C uses signed-distance selection-sort comparisons;
- C9F68 wraps the shifted numerator before signed division and then clamps the signed denominator/result.

A historical replacement deliberately returned `-EDOM` on zero normalization references instead of reproducing the stock divide/assert path. Keep that distinction explicit: the arithmetic above is reverse evidence, while survival/error policy belongs to the replacement implementation.

Manual AWB rollback in the historical owner also had to snapshot the complete affected generation/state and reject restoration after profile/generation mismatch. This is useful replacement-owner policy, but external sensor rollback is not atomic hardware rollback and must not be labeled as stock semantics.
