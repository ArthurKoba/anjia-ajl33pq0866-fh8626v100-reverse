# Roadmap

## Phase 1 — related repository integration

Status: **NEXT**.

Prepare and reconcile the repositories that own implementation changes:

- Builder/device profile;
- Divinus reference implementation;
- Firmware packaging/platform integration;
- Linux/kernel platform work;
- U-Boot where still required.

Use the camera contracts in this repository as the technical authority and keep implementation changes in the component repository that owns them.

## Phase 2 — Majestic reproducible baseline

- Pin the exact candidate binary/build provenance.
- Pin runtime libraries and canonical configuration.
- Validate VI, VENC and sustained RTSP independently of ISP quality.
- Validate ISP/color/exposure/day-night after media stability.
- Validate capture/audio/two-way audio separately.

## Phase 3 — product integration

- Move validated implementation changes into the repository that owns each component.
- Keep camera-level facts and contracts here.
- Keep Builder as the product-image/device-profile assembly boundary.
- Preserve `/dev/fh_pwm` as the accepted PTZ backend and complete higher-level calibration/autotracking.
- Keep evidence classes explicit: static/source coverage does not imply hardware acceptance.

## Phase 4 — upstream curation

When a component is ready for upstream contribution:

- start from the verified current upstream base;
- curate only accepted changes;
- build a small coherent commit series;
- verify build and target behavior at the appropriate evidence level;
- leave final pull-request creation to the repository owner.

## Reverse boundary

Broad reverse is not a project phase. Ghidra MCP is a standing capability used on demand for focused implementation or validation questions.
