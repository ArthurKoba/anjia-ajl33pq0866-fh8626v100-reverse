# FH8626 stock ISP writer map

This is the normalized implementation-facing map recovered from the closed stock reverse. It is a routing aid for focused future work, not a request to reopen broad ISP reverse.

Evidence class: `REVERSE_CONFIRMED` unless a stronger target class is cited by a subsystem document.

## Writer / controller map

| target identity | stock role | captured/default DAY state | durable implementation note |
|---|---|---|---|
| `D0FEC` | NR3D | active | current NR3D profile/state and hardware writer |
| `D0E5C` | NR2D | active | writes `ISP +0x454..+0x460` and low16 `+0x464` |
| `D2074` | YNR | active | writes `+0x4F4/+0x4F8/+0x4FC/+0x514/+0x518/+0x51C` and `+0xA0C..+0xA68` |
| `CEACC` | DPC | active/profile-gated | profile-controlled DPC hardware state |
| `CDD6C` | APC/detail/edge | active | current-DAY arithmetic/tables close the `+0x528..+0x574` family |
| `CE7D8` | CNR | active/profile-gated | writes `+0x520/+0x524` |
| `D1258` | purple-fringe suppression | profile-gated | LUT-dependent writer; instruction-level detail remains in canonical Ghidra MCP |
| `D0238` / `CFEB0` | LC / coefficient path | active/profile-gated | descriptor/arithmetic/packing contract retained; mutable detail remains in Ghidra MCP |
| `CECF0` | false-color | active/profile-gated | owns `ISP +0x4B8[31:24]` |
| `D1724` | RGBA controller | disabled in captured DAY profile | alternate-profile/full-parity oracle retained in Ghidra MCP and durable current contracts |
| `D1DB0` | YC | active | owns `+0x5C8/+0x5CC/+0x5D0` and `+0x4DC..+0x4E8` |

## Ownership rule

Do not collapse these writers into one untyped ISP register replay layer. Shared registers require typed ownership and masked updates; coherent color publication additionally follows the generation/ownership rules in `color-hal-contract.md`.

Detailed instruction-level arithmetic, xrefs, decompiler state and working annotations for these routines live in the canonical Ghidra MCP project. Git retains the durable writer identities, ownership rules and implementation-facing contracts. No Git-side `reverse/` report tree is expected or authoritative.

## Reopen policy

Broad stock writer discovery is closed. Reopen one writer in Ghidra MCP only when a current Majestic/owner integration blocker names that writer or contradicts its recovered contract.
