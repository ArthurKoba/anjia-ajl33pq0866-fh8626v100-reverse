# Stock human-detection contract

This document promotes the durable clean-room boundary from the completed 2026-08-31 static reverse of Apollo's human-detection path. Evidence is `REVERSE_CONFIRMED`; it is sufficient for an OpenIPC adapter/backend interface but does not grant redistribution rights for vendor code or model blobs.

## Architecture

The stock person detector is separate from the FH8626 BGM motion block:

```text
VPSS/profile 2
 -> Y8 extraction/copy
 -> HDT wrapper
 -> statically linked ARM OBJDETECT engine
 -> per-model results
 -> 32-byte HDT callback
 -> application event state
```

No separate detector/NPU device or accelerator ioctl was identified on this path. The classifier is statically linked ARM software (`OBJDETECT V2.1.0(g57c0608)`, build 2021-05-10).

## Input contract

Canonical detector input:

- source/profile index 2;
- visible 640x360;
- coded 640x368;
- stride 640;
- continuous 8-bit Y8;
- maximum source cadence 5 fps;
- stock analytics bitrate profile 131072;
- key interval 30.

The worker obtains a VPSS frame through the recovered channel-frame boundary and immediately materializes/copies Y8. Treat the upstream descriptor as borrowed/driver-managed and retain/copy the luma plane before the source can be recycled or reused.

## Model set

Stock model directory: `/app/hdt_model`.

Exact supplied human blobs:

- `model_human_86185422.bin.lzma` -> `headshoulder`, threshold 15;
- `model_human_86185770.bin.lzma` -> `person`, threshold 25.

The static model table also exposes a `face` type entry, but no exact face model blob/model ID is present in the supplied set. A `vehicle_onoff` control surface exists, but no matching backend/model/result path was proved. Therefore:

- human/head-shoulder backend contract: proved;
- face-capable path: proved, deployable exact stock model artifact not proved;
- vehicle: control-only/unproved capability.

## Result ABI

Each model result is exactly 0x58 bytes:

```c
struct objdet_box10 {
    int16_t x;
    int16_t y;
    int16_t width;
    int16_t height;
    uint16_t confidence;
};

struct objdet_model_result88 {
    struct objdet_box10 box[8];
    uint32_t count;             /* +0x50, clamped to 8 */
    uint32_t reserved_or_pad;   /* +0x54 */
};
```

Earlier wording that called `+0x54` a model ID is superseded.

## Callback ABI

The detector dispatches:

`callback(event_id, payload, 32, opaque)`

Stable payload words are:

- `+0x00` u32 output width;
- `+0x04` u32 output height;
- `+0x08` s32 bbox x;
- `+0x0c` s32 bbox y;
- `+0x10` s32 bbox width;
- `+0x14` s32 bbox height;
- `+0x18` u32 opaque auxiliary word A;
- `+0x1c` u32 opaque auxiliary word B.

The final two words are mode-specific state with no stable public semantic name proved. Preserve them opaquely. Likewise, several numeric event IDs must remain numeric; older guessed semantic labels are not authority.

## Interval semantics

Persisted `human_interval` reaches the worker control at the recovered target address. The exact worker contains a special `value == 1` branch that enforces approximately 2000 ms between detections. Other values bypass that specific gate.

Do not generalize the stored value into an arbitrary millisecond interval from help text alone.

## Closure

Static reverse is exhausted for an OpenIPC human-detector adapter. Remaining work is implementation/backend selection, target integration and legal/model availability rather than an unknown HAL ABI.

The distinct BGM foreground-motion engine is documented in `bgm-motion.md`; it must not be presented as person recognition.
