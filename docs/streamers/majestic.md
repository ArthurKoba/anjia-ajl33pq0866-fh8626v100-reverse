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
