# Color HAL ownership and generation contract

This document promotes the durable implementation contract from the closed stock color/HAL reverse. It is not a tuning guide: the primary requirement is coherent state ownership and publication order.

## Processing chain

The closed stock model is:

`GC1054 RAW10 -> VI/ISP numeric format -> Bayer/orientation -> fixed ISP init -> BLC -> early CFA/demosaic -> Bayer-sensitive AWB gains/state -> AWB state -> C9F68 selectors -> CCM -> GB/FC -> YC -> gamma/LUT -> downstream IQ/output`.

Physical RAW/CFA facts:

- 1280x720 RAW10, 1600 bytes/row;
- WIDE and TELE share the same physical format;
- normal RGGB; mirror GRBG; flip GBRG; mirror+flip BGGR.

## Fixed-init publication boundary

A 2026-09-05 current-Ghidra execution audit of stock `C4998` proves that its initialization prefix is order-sensitive and contains **236 ordered register stores** before the `C48CC` table-initialization call. Forty transient writes in the `0x270..0x30c` region are real execution steps even where later writes overwrite the same registers; they must not be removed merely because the final register image looks identical.

The exact execution order also places late writes to `0x1f0/0x084/0x088/0x068` and early writes to `0x43c/0x440/0x444/0x448`. The historical replacement had incorrectly reduced/sorted this sequence mainly by address and had placed the `0x178/0x024/0x028` tail inside the prefix. The corrected contract keeps that tail after `C48CC` in the known-stock-init orchestrator.

This prefix result was checked by executing 432 actual ARM instructions in a strict synthetic MMIO/RAM emulator and comparing the complete fake MMIO aperture against all 236 expected writes. It is `REVERSE_CONFIRMED` for instruction/order equivalence, not physical image-quality acceptance. The later initial 80-word gamma binding remained unresolved in that batch; do not claim full stock init from the prefix alone.

## Shared-register ownership

Key register ownership recovered from stock:

| surface | owner / semantics | replacement rule |
|---|---|---|
| `ISP+0x024` | shared Bayer/input-format, several feature bits and cold-init fields | masked RMW only after cold init |
| `+0x084/+0x088` | BLC | typed BLC writer |
| `+0x224/+0x228` | AWB Bayer-position gains | publish from persistent AWB state |
| `+0x330/+0x334` | GB | typed GB writer |
| `+0x3AC/+0x3B4` | format/orientation fields | masked RMW |
| `+0x4B8[31:24]` | false-color control | masked byte write |
| `+0x4C0..+0x4D4` | CCM | publish one complete CCM generation |
| `+0x4DC..+0x4E8` | YC coefficients | typed YC writer |
| `+0x5C8/+0x5CC/+0x5D0` | dynamic YC | typed YC writer |
| `+0x528..+0x574` | APC/detail family | separate feature-gated module |
| `+0x1000/+0x1180/+0x1300/+0x1480` | LUT/gamma/profile banks | profile generation ownership |

The historical normal baseline value `0x0A61EB00` for `ISP+0x024` must not be replayed as a universal runtime full-word assignment.

## Orientation and Bayer generation

Stock keeps ISP Bayer phase coherent with physical sensor orientation. The replacement must preserve this ordering and invalidate only replacement-owned caches that actually depend on the old Bayer generation.

Do not collapse the distinct stock orientation tables into one guessed generic table.

Pure WIDE/TELE switching does not require sensor format reinitialization because both prepared GC1054 targets share the same physical format.

## AWB -> CCM atomic dependency

The durable stock dependency is:

`persistent S0/S1/S2 -> +0x224/+0x228 -> A8/AA ratios -> C9F68 B0/B1/B2 selection -> CE670/CE628 CCM words -> +0x4C0..+0x4D4`.

C9F68 does not read the visible AWB gain registers directly. Therefore writing only `+0x224/+0x228` cannot reproduce the stock color transaction.

A replacement should treat one accepted AWB generation as containing the visible gains, ratio fields, selector state, CCM coefficients and final six-word CCM publication. Do not report AWB committed if only the visible gain registers changed.

### Exact CCM working-context semantics

A later current-Ghidra audit recovered two implementation details that are easy to lose in a high-level rewrite:

- `CE670` masks the interpolation weight to 12 bits; stock does **not** clamp it to 256. Values above 256 therefore make the `(256-weight)` term signed-negative for extrapolation;
- the six resulting CCM words are written to working context `ctx+0x140..+0x157` in loop order first, and only then does `CE628` publish that context to `ISP+0x4C0..+0x4D4`.

A historical replacement had fused interpolation directly into MMIO and skipped the working-context state. That is not stock-equivalent and breaks snapshot/rollback/generation coherence.

## Periodic error-continuation boundary

Stock periodic processing is not a blanket “abort on first ISP substage error” pipeline. Current-Ghidra inspection showed:

- `CB970` does not branch on the return status of the `C949C`/AWB/late-stage calls that were audited;
- `CED28` returns/stores its own status and only the exact success value `1` gates the gamma ioctl path;
- later independent LTM/purple/LC/FC/RGBA stages still execute after that local gamma decision.

A replacement may deliberately survive invalid-provider states more safely than stock, but it should isolate dependent work rather than accidentally suppress every independent downstream stage. This is a control-flow contract, not permission to ignore startup, sensor or hardware errors globally.

## Generation / invalidation model

Use separate monotonic generations where relevant, including:

- owner;
- sensor;
- sensor format;
- orientation;
- Bayer;
- geometry;
- profile;
- scene;
- statistics;
- AWB;
- IQ publication;
- stream.

Key rules:

- lens switch changes sensor/stream generation as appropriate but does not recreate sensor format/profile state;
- profile/scene updates rebuild only their dependent state and wait for fresh statistics before dynamic commit;
- one automatic AE/AWB decision consumes one unique immutable statistics generation;
- never drive AE/AWB from encoded-frame dequeue count;
- replacement publication may be stronger than stock: build a private candidate, verify dependency tags, publish in order, then advance IQ generation;
- partial failure should roll back or poison the owner rather than silently exposing mixed generations.

## Stock per-frame order

Among enabled blocks, preserve the established dependency order:

`AWB -> BLC -> NR3D -> NR2D/detail-adjacent stage -> CCM -> GB -> YC -> gamma candidate -> downstream IQ/APC -> staged publish -> LTM -> later coefficient/output controllers`.

Exact private function names/addresses remain provenance, not portable API.

## Green-cast conclusion

The strongest historical replacement defect was not an unknown sensor CFA or a large BLC pedestal. It was an incomplete coherent AWB -> selector -> CCM transaction, compounded by unsafe shared-register ownership and stale replacement generations.

Do not tune CCM/YC by eye before ownership/publication coherence is correct.

## Evidence boundary

Broad color/HAL reverse is closed for implementation. Reopen only if a current Majestic/owner integration blocker contradicts these contracts.
