# Historical Divinus engineering findings

This file preserves source-level lessons from the FH8626V100 Divinus work. Current product status is `../../docs/streamers/divinus.md`; implementation belongs in `ArthurKoba/openipc-divinus`.

## External encoded-source path

The first FH8626 integration used an external encoded H.264 source above hardware-HAL ownership. Host tests proved synthetic FH86 -> Divinus -> RTSP/TCP -> RTP/H.264 end-to-end without inventing an unproven native hardware ABI.

This was host/source evidence, not physical FH8626 acceptance.

## Confirmed native source-parity issues

Historical native-HAL work exposed several `SOURCE_CONFIRMED` mismatches:

- ISP runtime-bank/statistics-root handling diverged from the owner behavior;
- frontend barrier fields were accessed through a different address domain;
- candidate frame-wait behavior was introduced without an established target contract;
- RTSP descriptor/parser ownership regressed relative to the working implementation.

These are source mismatches and architecture risks, not automatic proof that every build fails on hardware.

## Capture-clock and bounded-send correction

A measured 15-second sample once showed fMP4 video advancing only about 10 seconds while audio advanced about 14.976 seconds. The cause was fixed-duration video timing derived from configured FPS instead of the actual source clock.

The corrected source model used source microsecond timestamps, reset timing state across generation changes, rebased invalid/backward deltas conservatively and shared each encoded access unit consistently between publication paths.

HTTP and RTSP/RTCP writes were also bounded by a total send deadline so a non-draining client could not hold publication indefinitely.

A later target probe showed MP4 video and audio advancing close to wall time, and VLC playback was reported normal at 1920x1080 / roughly 16.66 fps. This was positive historical target evidence, not long-duration/fault-path acceptance.

## Random-access epoch defects

Historical source review identified a separate class of H.264 lifecycle defects:

- lens-switch warm-up frames could be published before they were later discarded locally;
- reconnecting clients could receive an arbitrary mid-GOP access unit;
- generation change did not automatically invalidate downstream decoder/segment state;
- encoder refresh/restart was not transactionally tied to a guaranteed SPS/PPS/IDR boundary;
- a pending access unit could cause the first unit of a new encoder epoch to be dropped.

Durable requirement: lens change, encoder restart and reconnect each establish a new random-access epoch. Drop warm-up before publication, invalidate prior decode state, wait/request SPS/PPS/IDR and only then expose the new epoch downstream.

## RTP access-unit timestamp defect

Three target H.264 captures showed legal encoder POC behavior; the encoder did not justify a POC workaround.

Old RTP source code instead refreshed a connection timestamp only on marker packets, allowing different packets of one H.264 access unit to carry different timestamps.

The source correction selected one RTP timestamp once per access unit before emitting any NAL/FU-A packet. Future validation must enforce exactly that invariant.

## Invalid listener-attribution test

One later target test was invalid because an older Divinus process still owned the RTSP listener while the new candidate failed bind. VLC therefore exercised the older process.

Durable rule: verify candidate path/hash, PID/executable, listener ownership and ready state before interpreting any stream result.

## Work boundary

Future Divinus experiments should keep four concerns separate:

1. sensor/media initialization;
2. ISP/color control loop;
3. encoder ownership/lifecycle;
4. RTSP transport/reconnect/timestamp behavior.

Do not use this historical file as current implementation authority. Camera-level contracts remain in this repository; source changes belong in `openipc-divinus`.
