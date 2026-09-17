# Stock video timing, geometry and encoder ABI

This document promotes the durable timing/rate-control/output-geometry findings recovered from stock Apollo, `enc.ko`, `isp.ko`, `media_process.ko`, runtime UART traces and target H.264 captures. Evidence is primarily `REVERSE_CONFIRMED` plus bounded target-runtime validation.

## Proven cadence boundary

The exercised stock pipeline has different clocks at different layers:

- GC1054 / ISP: approximately 25 fps;
- public VPSS / VENC streams: approximately 16.66 fps;
- private analytics channel: 5 fps.

Therefore encoded-frame dequeue cadence is not the sensor/ISP control cadence. Do not drive AE/AWB from VENC dequeue timing, and do not infer a 1080p sensor mode from the 1920x1080 public stream.

Exact frame-drop placement, jitter, encoder timestamp origin and the relation between ISP statistics epochs and VENC dequeue remain runtime questions unless a narrower target trace proves them.

## Stock output topology

The native sensor/ISP geometry remains `1280x720`. Stock scaling/output selection occurs downstream in the VPU/VPSS layer.

Recovered stock channels:

- channel 0 public main: target `1920x1080`, nominal `16.66 fps`, scaler selector `13`;
- channel 1 public sub: target `640x360`, nominal `16.66 fps`, scaler selector `3`;
- channel 2 private analytics: target `640x360`, `5 fps`, scaler selector `3`; human detection consumes a VPSS-side Y8 frame rather than decoding its H.264 stream.

Stock startup order places target width/height in VPSS/VPU configuration before VENC creation, then starts/binds VENC and finally applies the scaler selector. This is direct evidence that FullHD is a downstream output feature, not a GC1054/RAW-mode change.

JPEG is a separate path; the retained stock snapshot geometry is `640x384` and must not be conflated with the `640x360` H.264 substream.

## Exact bounded startup geometry recovered in 2026-09-05 audit

The later focused `isp.ko` / `enc.ko` / `media_process.ko` audit closed the bounded startup geometry needed by the historical owner for 360p/720p/1080p output. The implementation used visible geometry, aligned scaler surface and a separate allocation envelope:

| visible | scaler surface | allocation envelope | X step | Y step | selector |
|---|---|---|---:|---:|---|
| `640x360` | `640x368` | `640x384` | `0x200000` | `0x1f4dea` | `3` |
| `1280x720` | `1280x720` | `1280x736` | `0x100000` | `0x100000` | inherit |
| `1920x1080` | `1920x1088` | `1920x1088` | `0x0aaaab` | `0x0a9697` | `13` |

For 1080p, the final eight rows to 1088 are part of the aligned scaler/coded surface, not merely spare allocation. This does **not** prove a CPU-linear NV12 stride/plane layout.

The exact startup sequence recovered for the bounded path includes VPU memory query/system-memory setup, three-word VI attributes `{1280,720,0}`, channel memory query/set, channel config `{chn,width,height}`, channel open, ISP start, PAE setup and VPU enable. Crop/rotation/hot-resize and full multi-channel runtime ownership remain separate concerns.

PAE/VENC receives visible dimensions and generates SPS/PPS/crop information itself. The replacement owner therefore did not need an invented custom SPS crop writer.

## Bounded 1080p target validation

A temporary historical owner build was exercised on target with `FH8626_OUTPUT=1080p`.

Positive observations:

- startup reported visible `1920x1080`, aligned surface/envelope `1920x1088`;
- SPS encoded macroblock height 1088 with bottom crop producing visible 1080;
- RTSP/MP4 decode completed in bounded tests;
- one 12-second sample decoded 198 frames with no dup/drop and an interior cadence corresponding to approximately `16.6605 fps`;
- SPS timing (`time_scale=90000`, `num_units_in_tick=2701`) also corresponds to approximately `16.6605 fps`.

This is a bounded `TARGET_RUNTIME`/partial acceptance point, not full stock-mode certification. The scaler-coefficient physical readback attempt produced `0/512` matches and was explicitly **not accepted** because the read/clock contract was unresolved. No blind coefficient rewrite was attempted. 360p/720p target acceptance, hot resize, crop/rotation/substream behavior and long-duration visual/motion acceptance remained open in that historical run.

## Stock channel rate-control profiles

Recovered stock profiles include:

- channel 0 public main: 1920x1080 H.264 Main, nominal 16.66 fps, AVBR, requested bitrate 786432, observed internal rate 707788, init QP 38, key interval 100;
- channel 1 public sub: 640x360 H.264 Main, nominal 16.66 fps, AVBR, requested bitrate 393216, observed internal rate 353894, init QP 35, key interval 100;
- channel 2 private analytics: 640x360, 5 fps, requested/internal rate 131072, key interval 30.

These are stock runtime/configuration facts, not recommended product defaults for Majestic.

## Encoder configuration boundary

`pae_enc_set_config` accepts an 11-word / 0x2c-byte channel configuration record. Exact recovered facts include geometry, codec/profile selector and initial QP, but the record does **not** prove bitrate, max-bitrate, GOP/key interval or min/max-QP semantics for the unlabelled words.

The channel must be stopped for this configuration path. On success the driver validates geometry/format, copies the record, regenerates SPS/PPS, reinitializes firmware registers, clears ROI/smart state and unregisters/re-registers the media source.

Treat it as stopped-channel reconfiguration, not an atomic transactional update. Production code must quiesce/drain outstanding stream leases and retain the complete previous configuration for recovery.

## Separate rate-control ioctl

The missing RC surface is separate from `PAE_SET_CONFIG`:

- `0xC054502F` -> `pae_set_rc_cfg`;
- `0xC0545030` -> `pae_get_rc_cfg`.

The user record is 0x54 bytes: channel word plus twenty 32-bit RC words. The setter validates through the stock RC core, stores the complete 0x50-byte payload in channel state and regenerates SPS/PPS.

Exact field positions proved by the stock body include:

- `rc_mode`;
- packed frame-rate count/time word;
- initial QP;
- primary rate/bitrate input consumed by the RC core;
- I/P min and max QP;
- I/P proportion fields;
- fluctuation level;
- signed IP QP delta;
- AVBR still-rate percentage;
- max-rate percentage;
- max-still-QP;
- several additional fields whose semantic names remain intentionally raw.

Do not invent units for unproved fields or derive hidden max-bitrate semantics merely from observed internal rates.

The dispatcher also exposes bounded realtime RC change at `0xC01C5055`, which updates frame-rate and I/P QP bounds without replacing the full RC block. `pae_set_rc_ctrl` separately updates advanced RC words and forces an I-frame.

## Transport ownership boundary

`media_process.ko` exposes a separate encoder-stream FIFO: 128 entries, 0x168-byte records and four-word ring metadata. That proves ownership/transport structure only; it does not make FIFO dequeue cadence equivalent to sensor or ISP cadence.

Existing H.264 captures support legal GOP-local POC reset behavior and observed IDR counts. They do not by themselves prove all timestamp relationships.

## Current use

This contract is retained primarily as an oracle for any Fullhan media backend or Majestic compatibility work. Current product strategy remains Majestic-first, and a future implementation should copy only semantics that are actually required and target-validated.
