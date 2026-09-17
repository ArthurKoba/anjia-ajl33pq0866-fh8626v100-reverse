# Media pipeline and streaming findings

This page contains the current camera-level media contract. Detailed historical experiments are retained under `history/media/` and do not define present acceptance.

## Stock pipeline

`STOCK_RUNTIME`: the stock application exposes a 1920x1080 main H.264 stream and 640x360 secondary stream while the exercised native GC1054 mode is 1280x720. Therefore downstream scaling exists; 1080p output does not imply a native 1080p sensor mode.

Sensor/ISP cadence, public VPSS/VENC cadence and analytics cadence are distinct. Exact cadence/rate-control details are documented in `video-timing.md`.

## Motion and human detection

Two stock paths are distinct:

- `bgm.ko` / `/dev/bgm` — hardware-assisted background/foreground motion-map path (`bgm-motion.md`);
- Apollo HDT/OBJDETECT — Y8 input plus person/head-shoulder classifier (`human-detection.md`).

BGM is not an audio device and does not classify a moving region as a person.

## ARC video/RPC

Focused reverse of `rtthread_arc.bin` recovered the `v_score` / `v_venc` service objects, shared control dispatcher, RPC/vbus/transport chain and video command branches `0x4301..0x4304`. The durable static contract and runtime-object boundary are in `arc-video-rpc.md`.

This is reverse evidence from ARC firmware, not proof of a complete portable userspace ABI.

## Streamer-independent transport constraints

Historical owner/Divinus work established several durable transport rules:

- source/capture timing must drive downstream timestamps; configured FPS alone is insufficient;
- one H.264 access unit uses one RTP timestamp across all packets/fragments;
- reconnect, lens-generation change or encoder restart establishes a new random-access epoch and must not expose arbitrary mid-GOP state;
- slow/dead clients must not hold shared publication/media locks indefinitely;
- target tests require candidate attribution: executable identity, PID, listener ownership and ready state.

These are implementation constraints, not acceptance of any particular streamer build.

## Divinus boundary

Divinus remains a reference path. Source-level transport/parity issues are summarized in `../streamers/divinus.md`. A renewed target candidate requires fresh listener/PID/candidate-identity validation and independent media/ISP acceptance.

## Majestic boundary

Majestic is the preferred product path. Required acceptance order is:

1. VI;
2. VENC;
3. sustained RTSP;
4. ISP/color/day-night;
5. audio/restart/reconnect as applicable.

A process start alone does not satisfy the media gates.

## Reverse boundary

Current media contracts stay in Git. Any new low-level media reverse is performed through the canonical Ghidra MCP project only when a concrete implementation or validation blocker requires it.
