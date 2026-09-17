# Historical AE engineering observations

This file preserves point-in-time AE observations that are useful for interpreting current contracts. It is not current authority; use `../../docs/isp/ae-controller.md` and `../../docs/isp/ae.md` for present implementation and acceptance boundaries.

## Lens-switch exposure recovery lesson

Historical native/runtime candidates deliberately separated AE into observation, physical writers and automatic-controller gates. During WIDE -> TELE -> WIDE testing, a replacement owner could revisit WIDE with transient exposure/gain state when no automatic statistics provider restored the previous lens state. A later candidate cached per-lens integration/gain and restored it on revisit when no provider existed.

That recovery behavior was replacement-owner policy. It must not be described as a proved stock atomic lens-switch contract.

## Temporary owner observation

A 2026-09-05 operator observation for temporary owner candidate `a7f208cd` reported normal/improved brightness and removal of green highlights while white clipping remained. Automatic AE was still classified as partial at that point, with initializer/state-provider/history parity work outstanding.

Treat this as scene-specific historical feedback, not controlled stock-vs-production AE acceptance.

## Apollo AE runtime slice

Historical identity:

- SHA-256 `6d8f6ab1ee7b2b797ddb1b5cfe0d6aa3313a414c1ff145cffeae150c52c003e6`
- historical filename `6d8f6ab1__apollo_rw_312cc8_31d070.bin`
- recorded size: 41,896 bytes

The captured process-memory region contained AE-state labels including `AE_INIT`, integration-time, sensor-gain, ISP-gain, iris and antiflicker adjustment states together with GC1054/media/ONVIF runtime strings.

This was reverse substrate only. It did not establish portable structure boundaries, stable addresses, control-flow ownership or a reusable AE ABI.

The object is not part of the selected current external-evidence manifest. Its SHA is therefore provenance-only unless the bytes are deliberately re-retained and a durable locator is added to `evidence/MANIFEST.tsv`.

## AEV1 stripe captures

Historical primary identities:

- `2876329eaa7d75e635c17033fcb588f7a6372be84cdd4278ff24e0c58841837d` — `aev1_5s.h264`
- `3348d99a0ec85979e7147bd3605a3fb21212444a9deae8acc2b4aca79b424591` — `aev1_transition_5s.h264`

Both were recorded as 1280x720 H.264 Baseline captures with 125 frames at an effective 25 fps. Visual review showed a dark green-dominant failure state with horizontal stripe/line corruption. The transition capture showed the same artifact class while scene intensity/geometry changed.

These captures proved the historical image failure existed; they did not identify which AE/ISP block caused it. Historical MP4/PNG members were derivatives of the raw captures and did not add unique image evidence.

These SHA identities are retained here for provenance only. They are not current evidence dependencies unless a corresponding row with durable locator exists in `evidence/MANIFEST.tsv`.

## Current-use rule

Historical SHA identities may explain how a contract was reached, but current implementation or acceptance claims must depend on current Git contracts, canonical Ghidra MCP state, or external objects indexed in `evidence/MANIFEST.tsv`.
