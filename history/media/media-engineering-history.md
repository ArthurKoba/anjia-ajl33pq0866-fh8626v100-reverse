# Historical media pipeline and streaming findings

This file preserves point-in-time media/streaming observations. Current authority is under `docs/media/` and `docs/streamers/`.

## Stock pipeline

Stock application reported a 1920x1080 main H.264 stream and a 640x360 secondary stream while the exercised native sensor mode was 1280x720. Therefore stock performs downstream scaling; 1080p output does not imply a native 1080p sensor mode.

The durable cadence and encoder/rate-control boundary is `../../docs/media/video-timing.md`: sensor/ISP cadence, public VPSS/VENC cadence and analytics cadence are distinct domains.

## Motion and human detection

Stock contains two separate paths:

- `/dev/bgm` / `bgm.ko`: hardware-assisted background/foreground motion processing, documented in `../../docs/media/bgm-motion.md`;
- Apollo HDT/OBJDETECT: Y8 input plus statically linked ARM person/head-shoulder classification, documented in `../../docs/media/human-detection.md`.

BGM is not an audio device and does not classify a moving region as a person.

## ARC video/RPC reverse

Focused analysis of `rtthread_arc.bin` recovered the `v_score` / `v_venc` service objects, common control dispatcher, RPC/vbus/transport chain and video command branches `0x4301..0x4304`.

The durable contract and its runtime-state limits are in `../../docs/media/arc-video-rpc.md`. Static reverse does not by itself prove a complete portable userspace ABI.

## 2026-09-05 owner geometry checkpoint

A temporary owner candidate reached a 1080p target surface and passed the exercised SPS/decode path. At that point scaler-coefficient readback remained unresolved (`0/512` was recorded), image/DDR/all-preset acceptance was open and full cadence/rate-control parity was not established.

This result must not be promoted to native-owner hardware acceptance. The stronger durable stock fact is only that downstream scaling exists.

## 2026-08-25 OpenIPC media-stack observation

A historical OpenIPC checkpoint captured `gpio_wave`, `bgm`, `jpeg`, `enc`, `isp`, `media_process`, `xbus_rpc` and `vmm` loaded together. `/dev/mtdblock6` was mounted read-only on `/mnt/stockapp`; an anonymous VMM physical zone at `0xA2700000..0xA3F7FFFF` was visible with no blocks allocated at capture time.

These are point-in-time observations, not guarantees that memory ownership/layout is identical on every boot.

## Historical Divinus external-source path

The external encoded-source path proved synthetic FH86 -> Divinus -> RTSP/TCP -> RTP/H.264 end-to-end on host. This was host evidence, not physical FH8626 acceptance.

A preserved working RTSP implementation used `dup()` for the write side and reset parser input state on connection reuse. A later native candidate diverged by wrapping the same fd multiple times and by not resetting parser input, creating a confirmed source-level regression.

Native FH8626 Divinus hardware acceptance remained incomplete. Current status is `../../docs/streamers/divinus.md`.

## Majestic acceptance boundary

Majestic validation must proceed VI -> VENC -> sustained RTSP with ISP excluded; ISP/day-night/color comes only after that base path is stable. Process startup alone does not satisfy these gates.

## Apollo runtime mapping observation

A stock interface probe showed Apollo's main executable mapping at `0x00010000..0x00303000`, another executable mapping at `0x00312000..0x00316000`, and an executable heap beginning at `0x00316000`.

A captured RW slice from `0x00312cc8..0x0031d070` contained ONVIF/WS-* schema strings, media-recording paths, the GC1054 sensor identifier and `/app/abin/ipc_def.db`. This ties stock Apollo writable runtime state to ONVIF/media/sensor-control material but does not establish individual field offsets or command ownership without narrower event-level evidence.
