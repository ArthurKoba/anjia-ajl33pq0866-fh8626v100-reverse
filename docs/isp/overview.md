# ISP overview

Broad stock reverse for the exercised pipeline is closed. Focused ISP work is resumed only for a concrete blocker.

The exact owner source is the primary implementation oracle below direct hardware evidence. FH8852-family code is structural reference only.

Detailed focused contracts already recovered and retained include:

- exact GC1054 day/night/wlight physical AE controller: `ae-controller.md`;
- NR3D gain/history/writer dependency: `nr3d.md`;
- shared-register/color generation ownership: `color-hal-contract.md`;
- current writer/register map: `writer-map.md`.

Do not reopen those areas broadly unless a current blocker contradicts the retained contract.

Frozen native Divinus currently has two generic ISP source-parity concerns in addition to AWB-specific work:

1. Frontend barrier: candidate accesses +0x18/+0x68-like offsets through ISP MMIO, while owner behavior uses runtime-context fields. Address domains differ.
2. Frame wait: candidate introduces `FH8626_ISP_FRAME_WAIT` / `0x40046908` timeout semantics not established by the preserved owner/reverse contract.

Do not treat either source mismatch as an automatic hardware failure; repair/justify first, then validate on target if native Divinus is resumed.

## Historical TELE day-to-night IRQ observation

One 2026-08-28 TELE day->night target capture has three IRQ snapshots. `before`: I2C0=43608, ISP=32184, PAE=8442, PWM=3120. `after_early`: I2C0=48286, ISP=34700, PAE=9349, PWM=3120. `after_settled`: I2C0=49577, ISP=35136, PAE=9463, PWM=3120. Total deltas are I2C0 +5969, ISP +2952 and PAE +1021 with PWM unchanged. This is observation-only evidence that the captured day/night transition produced substantial sensor/control and ISP/media activity without motor PWM activity. It does not identify the responsible calls or register writes.
