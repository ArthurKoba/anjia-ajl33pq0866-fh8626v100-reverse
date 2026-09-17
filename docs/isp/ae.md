# ISP AE findings

Historical native/runtime candidates deliberately separated AE into observation, physical writers and automatic controller gates. `aeauto` was not to be enabled before lower-level AE writer paths were established.

Historical lens switching exposed a practical AE-state problem: a stock-style WIDE->TELE->WIDE transaction could leave transient exposure at 64/64 when no automatic AE-stat provider restored the former WIDE value (historically observed as 208). A later candidate cached per-lens integration/gain and restored it on revisit when no provider existed.

This finding explains a historical failure mode; it is not a current accepted AE implementation. Any future native AE work must be tied to exact owner/runtime-bank contracts and target evidence.

## Exact current-day controller oracle

The closed stock GC1054 day-mode controller is normalized in `ae-controller.md`. It covers the statistics-ready cadence, nine-cell measured-value path, exact 60-frame history ordering, hysteresis/dwell gate, current-day integration bounds, actuator families, deferred `C6AC8` commit queue and GC1054 callback boundary.

Use that document as a focused implementation oracle only when a concrete AE/ISP blocker requires it. It does **not** establish stock-night parity, and Majestic-first work should not recreate the full stock controller preemptively.

## 2026-09-05 owner observation

The active corpus recovers operator feedback for temporary owner candidate `a7f208cd`: brightness became normal/improved and green highlights disappeared, while white clipping remained. At the same date the implementation status still classified automatic AE as `PARTIAL`, with remaining initializer/state-provider/history parity work; later undeployed source changes were explicitly outside that observation.

Treat this as scene-specific historical target feedback, not controlled stock-vs-production AE acceptance across lighting conditions. The fuller point-in-time context is preserved in `../../history/active-corpus-chronology-20260905.md`.

## Historical Apollo AE runtime slice

Archive payload SHA-256 `6d8f6ab1ee7b2b797ddb1b5cfe0d6aa3313a414c1ff145cffeae150c52c003e6` is a 41,896-byte point-in-time Apollo process-memory slice from the historical AE-runtime input set. It contains AE-state labels such as `AE_INIT`, integration-time, sensor-gain, ISP-gain, iris and antiflicker adjustment states, alongside GC1054/media/ONVIF runtime strings. The exact raw slice is retained in Drive as `6d8f6ab1__apollo_rw_312cc8_31d070.bin`.

Treat this as reverse substrate only: it proves those strings/state tables coexisted in the captured Apollo runtime region, but it does not by itself establish structure boundaries, addresses valid across boots, control-flow ownership, or a portable AE ABI. Credential-like runtime strings present in the raw capture are intentionally not copied into findings or general documentation.

## AEV1 stripe capture bundle

The historical `captures/aev1` bundle contains two primary 1280x720 H.264 Baseline captures, both 125 frames at an effective 25 fps: SHA-256 `2876329eaa7d75e635c17033fcb588f7a6372be84cdd4278ff24e0c58841837d` (`aev1_5s.h264`) and `3348d99a0ec85979e7147bd3605a3fb21212444a9deae8acc2b4aca79b424591` (`aev1_transition_5s.h264`). Visual review shows dark green-dominant imagery with horizontal stripe/line corruption in the captured failure state. The transition capture shows the same artifact class while scene intensity/geometry changes. The captures are failure evidence only; they do not establish which AE/ISP block caused the image defect.

The MP4/PNG members of the same archive are derivative rather than independent evidence. `aev1_5s_fixed.mp4` decodes to the same 125 frame hashes as `aev1_5s.h264`; the older `aev1_5s.mp4` exposes only 51 frames because of its broken/two-second container timing, but those decoded frames match the corresponding prefix of the raw capture. `aev1_transition_5s.mp4` has the same 51-frame timing/container limitation and its decoded prefix matches the transition raw H.264. Stripe PNG f001..f005 are exact ffmpeg-style frame extractions of raw `aev1_5s.h264` frames 3, 8, 13, 18 and 23. Therefore the two raw H.264 captures are retained as primary evidence and the MP4/PNG derivatives can be discarded without losing unique image content.
