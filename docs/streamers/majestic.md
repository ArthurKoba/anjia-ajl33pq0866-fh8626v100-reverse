# Majestic integration

Majestic is the preferred product path for this camera.

## Current known state

`OBSERVATION`: a retained FH8852-family Majestic build can start on FH8626V100 with the proprietary Fullhan userspace/media stack. Startup alone is not acceptance.

The current clean validation ladder is:

1. exact binary/build provenance;
2. runtime library set and configuration;
3. VI;
4. VENC;
5. sustained RTSP without ISP;
6. ISP/day-night/color;
7. audio/two-way audio;
8. restart/reconnect/regression behavior.

Do not claim a later gate from an earlier one.

## Porting rule

Majestic-specific work should consume camera-level hardware/media contracts from this repository rather than duplicating them as Majestic-only knowledge.

When an upstream/vendor Majestic build for FH8626V100 becomes available, record its exact binary SHA/build provenance and compare behavior against the current FH8852-derived baseline.


## Current compatibility surface

Firmware `work/fh8626v100-majestic@4bd2f8bc...` contains a source-built ABI probe for the retained FH8852V200 userspace baseline. It is deliberately non-mutating: no sensor, ISP, VI or VENC initialization is called. Its purpose is to separate three questions before the next adapter slice:

- are the eight donor libraries loadable as one closure under the FH8626 musl image;
- which expected Fullhan VMM/SYS/VPSS/VENC/MIPI/ISP symbols are actually resolvable;
- which native FH8626 device nodes are present in the clean image.

The implementation mapping is maintained in `docs/process/fh8626-majestic-staging.md`. The mapping reuses platform contracts from Divinus/reverse, but Majestic must own its own adapter and must not execute Divinus as a runtime dependency.

The donor Majestic package still fetches a moving `master` S3 artifact. The first new target run must therefore retain the runtime SHA-256 of the executable. No immutable donor SHA is currently indexed in the project evidence manifest.

## Upstream production request

The production request to Majestic maintainers is drafted in `docs/process/fh8626-majestic-upstream-issue.md`. It is not to be opened by an agent. The preferred production outcome is a native FH8626V100 platform build; the compatibility direction remains staging until either that exists or the source adapter boundary is fully understood, reproducible and hardware-accepted.


## Sensor compatibility adapter

The first media-facing source adapter is now staged. FH8852V200 sensor plug-ins use a 0x7c-byte callback object, while the recovered FH8626 GC1054 object is 0x68 bytes with materially different callback ordering. The old direct-plugin experiment therefore crossed a concrete ABI mismatch.

The new Firmware facade presents the FH8852 callback shape and translates the subset already proven on FH8626. It is reached only through the explicit `majestic-fh8626-media-run` path. The normal service remains media-off until the facade and the following ISP/VI/VENC layers receive fresh target evidence.


## Offline compatibility completion

The compatibility direction has now reached the offline software boundary. In addition to the sensor facade, Firmware contains source VMM and DSP compatibility layers, a fixed native FH8626 H.264 channel-0 bring-up path, stream-record translation with balanced native descriptor leases, and separate strict/permissive/native-video runners.

The control-plane metrics problem is fixed on the backend side: Majestic listens on loopback port 18080 and a small source proxy owns external port 80. The proxy forwards normal Majestic HTTP/API/WebSocket traffic and serves the existing stock `GET /metrics` contract from Linux `/proc` data. No WebUI JavaScript is patched. Temperature remains absent rather than fabricated.

Further offline wrapping is deliberately evidence-gated. The project already has native FH8626 ISP/image, JPEG, RTX audio, PTZ and illumination primitives, but their Majestic-facing APIs must be taken from an actual run or official Majestic platform contract rather than guessed from unrelated FH8852 structures.
