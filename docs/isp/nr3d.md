# NR3D runtime contract

This document preserves the durable target-exact NR3D dependency boundary recovered from the 2026-08-31 Apollo + stock `isp.ko` audit. Evidence class is `REVERSE_CONFIRMED` unless stated otherwise.

## Per-frame dependency chain

The important contract is not a standalone `SetNR3D` register write. Stock ordering is:

`C949C -> C73F8/C6D04 -> CB970 -> D0FEC`

where:

- `C949C` refreshes common control state before module writers;
- `C73F8` publishes current integration/gain state and packed total gain at `ctx+0x60`;
- `C6D04` refreshes the controller/history state;
- `CB970` is the periodic ISP pipeline;
- `D0FEC` is the NR3D writer when its gate is enabled.

A replacement must therefore keep the gain/history/control state live and ordered. A one-shot NR3D register replay with stale `ctx+0x60` is not stock-equivalent.

## Startup provider and mode readiness

A later 2026-09-05 current-Ghidra pass recovered a tighter startup boundary:

- `C1EE4` issues ioctl/request `0x80206926` into the context state at `ctx+0x11AC` before buffer sizing;
- `CB970` requires that mode value to equal `1` before calling `D0FEC`; feature-enable state alone is insufficient;
- `C1FB8` allocates two temporal buffers for nonzero mode before the statistics region. For the audited 1280x720 path the recovered sizes are `0x119400` and `0x0E100`, with statistics base `0x127500`, statistics size `0x42500`, and total required budget `0x16A400`;
- those offsets agree with the historical owner physical layout, but they do not by themselves prove complete allocation/init parity or every kernel-side reset semantic.

A replacement owner should therefore treat “mode query succeeded and returned the accepted runtime mode” as a separate readiness prerequisite rather than substituting a constant mode-1 assumption. Historical source code implemented this as an explicit 32-byte provider/query boundary and cleared NR3D eligibility on query failure; that is replacement safety policy, not proof of the exact stock error policy.

## D0FEC behavior

`D0FEC`:

- is called from the periodic pipeline when the relevant gate is active;
- obtains total gain through the common initialized `ctx+0x60` path;
- consumes direct/profile state around `ctx+0x1F0..+0x21C`;
- uses a signed coefficient multiplied by gain twice, with fixed rounding/saturation;
- publishes masked/RMW controls at mapped ISP offsets including `+0x44C`, `+0x450`, and `+0x468..+0x480`.

`C5DB8` returns `ctx+0x60 >> 12`, bounding the normal gain input to 20 bits. The audited D0FEC arithmetic keeps the signed high word through shift/add/rounding before clamp; historical replacement code that narrowed prematurely to signed 32-bit could wrap or change sign on ABI-valid extremes. This is an implementation-width requirement, not a reason to alter the stock formula.

No D0FEC instruction allocates, clears or rotates an NR3D temporal-history surface. History/state ownership is upstream of the final writer.

## Exact stock `isp.ko` boundary

The audited stock module was the exact 144,932-byte ARM EABI5 `isp.ko`.

Its proc parser proves:

- `nr3d_on`-style enable writes `1` to `(*fh81_isp)+0xA54`;
- a separate parsed one-bit field is stored at `(*fh81_isp)+0xCC0`.

These offsets are distinct controls. `+0xA54` must not be relabeled as a buffer pointer.

The module contains NR3D diagnostic strings around VPU memory setup, but the exact module does **not** prove that the associated extended allocation is the temporal NR3D history buffer or that it initializes NR3D coefficient/history lifecycle.

No target `API_ISP_SetNR3D` / `API_ISP_GetNR3D` function was identified in the current Apollo target database. Do not import homolog API names as target proof.

## Integration consequence

For a replacement owner, preserve:

1. valid per-frame common gain/statistics refresh;
2. correct C949C-before-CB970 ordering;
3. retained controller/history state across frames;
4. profile-dependent NR3D state;
5. valid runtime mode/provider readiness;
6. final masked/RMW publication.

Do not drive NR3D from an independent frozen gain word or from encoded-frame dequeue cadence.

## Scope

The stock NR3D dependency/writer contract is closed sufficiently for implementation/debugging. Exact vendor high-level API naming and the private temporal-buffer implementation are not required unless a current product blocker specifically depends on them.

Majestic-first work should use this as a focused oracle only if current NR3D/noise behavior becomes a concrete blocker; broad ISP reverse remains closed.
