# ARC video / RPC / vbus contract

This document records the durable findings from the focused reverse of `rtthread_arc.bin`. It is `REVERSE_CONFIRMED` static evidence, not a complete runtime ABI or hardware-acceptance result.

## Program boundary

The analyzed program is the ARC/RTThread image, not ARM Apollo. The retained image is ARCompact little-endian code loaded at `0xBFF80000`.

Ghidra pseudocode is not a substitute for instruction/byte verification when ARC hardware-loop or LIMM artifacts matter.

## Service and transport chain

Recovered static structure includes:

- video service registration for `v_score` and `v_venc`;
- a shared control dispatcher for both video services;
- RPC descriptor registration and runtime dispatch;
- transport callback slots for RPC and vbus;
- ring/semaphore/interrupt receive state;
- packet dispatch by transport selector;
- service-object control through the object control slot;
- callback packet generation including `H264_CB` and `HEVC_CB`.

Important recovered entry points include `0xBFF80234`, `0xBFF801E4`, `0xBFF8B1EC`, `0xBFF8B0DC`, `0xBFF8B294`, `0xBFF8B5B8`, `0xBFF8BDA0`, `0xBFF8BDB4`, `0xBFF8B5CC`, `0xBFF8B6C0`, `0xBFF8B5F0`, `0xBFF8B2A8`, `0xBFFA5A8C`, and `0xBFF8B534` for the analyzed build. These are build-specific reverse landmarks, not a public stable ABI.

## Video command surface

The dispatcher compares full 32-bit command words. Focused reverse recovered these branches:

- `0x00304301` (`low16 0x4301`): block-grid processing and result copy;
- `0x00A04302` (`0x4302`): map-buffer processing with cache/helpers;
- `0x01084303` (`0x4303`): H.264 path with block-grid work, packed/QP outputs and `H264_CB` notification;
- `0x00B44304` (`0x4304`): neighboring HEVC path.

Do not infer request payload size solely from the upper command bits; the command numbers alone do not prove a complete userspace layout.

## Runtime-state boundary

The static image does not contain the initialized runtime RPC descriptor arrays, service object envelopes, transport callback slots, shared-ring pointers or working map/table buffers. Concrete runtime values therefore require live target evidence or the active Ghidra project state; they must not be guessed from ARM/Apollo structures.

## Reverse authority

The canonical Ghidra project accessed through Koba MCP is the working reverse environment for this firmware. New function boundaries, types, xrefs, comments and focused follow-up belong there.

This repository keeps the stable ARC contract only. Do not recreate workspace-local Ghidra projects, export directories or generated per-function decompiler dumps in Git. Heavy primary inputs remain external evidence referenced through `evidence/MANIFEST.tsv`.

Reopen this area only when a concrete implementation or validation blocker requires a narrower ARC video/RPC question.
