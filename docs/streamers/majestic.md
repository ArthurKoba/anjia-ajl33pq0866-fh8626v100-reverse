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

Firmware `work/fh8626v100-majestic@718f6a14...` contains a source-built ABI probe for the retained FH8852V200 userspace baseline. It is deliberately non-mutating: no sensor, ISP, VI or VENC initialization is called. Its purpose is to separate three questions before the next adapter slice:

- are the eight donor libraries loadable as one closure under the FH8626 musl image;
- which expected Fullhan VMM/SYS/VPSS/VENC/MIPI/ISP symbols are actually resolvable;
- which native FH8626 device nodes are present in the clean image.

The implementation mapping is maintained in `docs/process/fh8626-majestic-staging.md`. The mapping reuses platform contracts from Divinus/reverse, but Majestic must own its own adapter and must not execute Divinus as a runtime dependency.

The donor Majestic package still fetches a moving `master` S3 artifact. The first new target run must therefore retain the runtime SHA-256 of the executable. No immutable donor SHA is currently indexed in the project evidence manifest.

## Upstream production request

The production request to Majestic maintainers is drafted in `docs/process/fh8626-majestic-upstream-issue.md`. It is not to be opened by an agent. The preferred production outcome is a native FH8626V100 platform build; the compatibility direction remains staging until either that exists or the source adapter boundary is fully understood, reproducible and hardware-accepted.


## Sensor compatibility adapter

The sensor path is now source-first. FH8852V200 sensor plug-ins use a 0x7c-byte callback object, while FH8626 uses a materially different 0x68-byte contract. Firmware presents the FH8852 shape but now backs it with source-native FH8626 GC1054 and source MIPI implementations recovered from the retained platform evidence. Gain, integration, timing, register I/O, format programming and mirror/flip are implemented from the recovered native contract. The normal service remains media-off until the new path receives target evidence.


## Offline compatibility completion

The compatibility direction has now reached the offline software boundary. In addition to the sensor facade, Firmware contains source VMM and DSP compatibility layers, a fixed native FH8626 H.264 channel-0 bring-up path, stream-record translation with balanced native descriptor leases, and separate strict/permissive/native-video runners.

The control-plane metrics problem is **not** papered over. Majestic continues to own its stock HTTP/API/WebSocket frontend directly. No JavaScript rewrite and no auxiliary HTTP proxy are accepted. The historical empty `GET /metrics` response remains a backend/provider blocker to fix in the real Majestic/platform integration.

Further offline wrapping is deliberately evidence-gated. The project already has native FH8626 ISP/image, JPEG, RTX audio, PTZ and illumination primitives, but their Majestic-facing APIs must be taken from an actual run or official Majestic platform contract rather than guessed from unrelated FH8852 structures.


The staging branch also contains a source RTX audio MPI facade. FH8852 `FH_AC_*` frame/config wrappers were statically matched to the same RTX command records already hardware-proven on FH8626, including AI frame/PTS and AO frame submission. This is source-ready but still requires Majestic runtime evidence for actual microphone, speaker and two-way ownership/policy.


## Final offline boundary

The clean-room Ghidra pass is complete enough to stop speculative implementation. The compatibility branch now uses source-native sensor/MIPI, VMM, VPSS/VENC/stream and RTX audio layers where concrete ABI mismatches were proved. It deliberately retains isolated donor ISP/ispcore where stock analysis shows a large shared userspace context/state machine and no concrete post-fix incompatibility has yet been observed.

Further changes to ISP/JPEG/audio policy should now be driven by the first real target run, not by guessed replacement code.
