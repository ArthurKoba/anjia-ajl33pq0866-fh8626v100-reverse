# JPEG snapshot and OSD contract

This document promotes the durable camera-level capability boundary from the closed stock application/HAL reverse. Evidence class: `REVERSE_CONFIRMED`. Runtime/visual behavior still requires product-specific validation where not separately hardware-proven.

## JPEG snapshot service

Stock JPEG is a separate snapshot service, not the public H.264 substream.

Recovered active configuration:

- channel: `1`;
- logical target: `640x384`;
- resize mode: `2`;
- QP: `76`;
- speed: `4`;
- buffer limit: `131072` bytes.

The `640x384` JPEG target must not be conflated with the stock `640x360` public video substream.

## Literal FH8626 JPEG/MJPEG ABI boundary

The 2026-09-05 deep owner/media audit established an important negative boundary. The retained corpus proved that stock loads/uses a JPEG path and that generic Divinus supports JPEG/MJPEG on other platforms, but it did **not** contain the exact FH8626 `jpeg.ko` ioctl ABI, request/result structures, quality/geometry controls, or concurrent H.264+JPEG ownership sequence.

Therefore:

- generic Divinus JPEG code is not evidence of the FH8626 hardware ABI;
- the historical H.264 sidecar v1 is Annex-B/session-state specific and cannot safely transport MJPEG merely by adding a codec flag;
- snapshot JPEG (request/response or latest-frame semantics) and continuous MJPEG (independently decodable frame stream with its own backpressure/epoch policy) must remain distinct;
- literal stock-equivalent MJPEG work should begin from the exact `jpeg.ko` plus stock ioctl/runtime trace, not from guessed structures.

This is an evidence-limit contract, not a claim that JPEG/MJPEG is unsupported by FH8626.

## OSD

Stock OSD starts through the vendor ADV_OSD service and registers multiple tag callbacks.

Recovered geometry demonstrates downstream alignment:

- logical `1920x1080` -> aligned `1920x1088`;
- logical `640x360` -> aligned `640x368`;
- native sensor/ISP geometry remains `1280x720`.

Therefore OSD is downstream of native RAW/ISP geometry. OSD implementation must not mutate RAW/CFA/AE/AWB ownership or treat aligned output height as native sensor height.

For OpenIPC, implement OSD against the native/open overlay path available to the selected streamer/platform. Do not preserve the proprietary ADV_OSD binary merely to reproduce its placement behavior.

## Related capability boundary

Stock product metadata also declares two public video streams, snapshot/JPEG, mirror/flip, IR/day-night, WDR control, microphone/playback and PTZ. Those capabilities are normalized in their owning subsystem documents; this file owns only JPEG/OSD semantics.

## Closure

No additional Apollo reverse is required for JPEG geometry, basic snapshot configuration or OSD placement before implementation. Reopen the **hardware-provider ABI** only for a concrete JPEG/MJPEG implementation blocker, using exact `jpeg.ko`/runtime evidence rather than generic cross-platform assumptions.
