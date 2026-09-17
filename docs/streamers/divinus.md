# Divinus integration

Divinus is the open reference and diagnostic implementation for FH8626V100. It is not the preferred product baseline; Majestic is the current product direction.

## Current role

Use Divinus to:

- compare an open implementation against the camera contracts preserved here;
- reproduce or isolate media/ISP/transport behavior when that helps current integration;
- carry selected FH8626 work into a clean implementation branch when the related-repository phase begins.

The implementation repository is `ArthurKoba/openipc-divinus`. This camera repository owns the hardware/media contracts, not the Divinus source tree.

## Known source-level parity gaps

`SOURCE_CONFIRMED` issues retained from the current source comparison:

- ISP runtime-bank/statistics-root handling differs from the retained owner behavior;
- frontend barrier/state fields were accessed through a different address domain;
- frame-wait behavior was introduced without an established owner-equivalent contract;
- RTSP fd/parser ownership regressed relative to the preserved working transport path.

These are source mismatches. They are not proof that every build fails on target, but they prevent treating the native path as accepted parity without repair or explicit justification and validation.

## Media/transport constraints retained from historical work

Future Divinus work should preserve these implementation constraints:

- source/capture timestamps, not configured FPS alone, drive downstream media timing;
- one H.264 access unit must use one RTP timestamp across all packets/fragments;
- reconnect, lens generation changes and encoder restart establish a new random-access epoch and must not expose arbitrary mid-GOP state;
- slow or dead clients must not hold shared publication/media locks indefinitely;
- candidate attribution must be proven before interpreting a target test: executable path/hash, PID, listener ownership and ready state.

The detailed experiments and obsolete candidate chronology are historical material, not current acceptance criteria.

## Acceptance boundary

A renewed Divinus target candidate needs independent validation of:

1. sensor/media initialization;
2. ISP/color behavior;
3. encoder ownership/lifecycle;
4. RTSP reconnect/timestamp behavior;
5. audio if included in the candidate.

Do not combine unrelated repairs into one diagnostic conclusion.

## Reverse boundary

If a Divinus blocker needs reverse analysis, use the canonical Ghidra MCP project. Do not recreate old local reverse/export workflows from historical Divinus notes.
