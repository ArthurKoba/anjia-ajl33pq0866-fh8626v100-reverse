# Current project state

Status: `ACTIVE / MAJESTIC_FIRST / DIVINUS_REFERENCE`.

Checked: `2026-09-17`.

## Authority

- GitHub owns current camera-level source, documentation, contracts, state and evidence manifests.
- Google Drive owns heavy or unique primary evidence referenced by SHA-256 from `evidence/MANIFEST.tsv`.
- Ghidra MCP through Koba MCP Bridge is the canonical mutable reverse-analysis workspace.

## Target hardware

- SoC: FH8626V100.
- Board/camera: ANJIA AJL33PQ0866.
- Sensors: dual GC1054 MIPI.
- Exercised native mode: 1280x720 @ 25 fps.
- WIDE is the product default lens.
- GPIO5 cold-boot sequencing is required for TELE visibility before media startup.
- Stock lens selection uses GPIO4/GPIO14 and coordinates switching with the media pipeline.
- `/dev/fh_pwm` PTZ control is hardware-proven on the exercised board after correcting swapped motor connectors.

## Sensor / ISP / media

Durable camera contracts are documented under `docs/sensor/`, `docs/isp/`, `docs/media/` and `docs/architecture/`.

Broad reverse of the exercised stock path is not an active objective. New reverse work should answer a concrete implementation or validation question in Ghidra MCP and then promote the durable result into current Git documentation or reusable source.

Static/source/reverse coverage is not equivalent to target runtime acceptance. Hardware acceptance remains explicitly labeled.

## Majestic

Majestic is the preferred product path.

A recent FH8852-family Majestic build can start on FH8626V100 with the retained proprietary Fullhan stack. The acceptance boundary is to pin a reproducible candidate and validate:

1. VI startup;
2. VENC startup;
3. sustained RTSP;
4. ISP/color/exposure/day-night behavior;
5. audio behavior.

Process startup alone is not media acceptance.

## Divinus

Divinus remains an open reference and diagnostic implementation.

Native FH8626 work reached the camera but is not the preferred product baseline. Known reference-path mismatches include ISP runtime-bank/statistics handling, frontend address-domain semantics, frame-wait behavior and RTSP fd/parser ownership.

## Audio

RTX microphone and speaker paths are hardware-proven on this board. Further audio work should be driven by concrete capture, playback or two-way-audio integration requirements.

## PTZ

The accepted low-level backend is `/dev/fh_pwm`. Remaining work is higher-level calibration and integration/autotracking rather than rediscovery of the motor backend.

## Curated source

Reusable camera-specific engineering components are retained under:

`source/fh8626v100/components/`

The tree contains independent board/control/diagnostic/media/platform/sensor components plus standalone media contracts and host tests. It is intentionally not a monolithic owner/application build. Production changes belong in the OpenIPC repository that owns each component.

## Evidence

The retained GC1054 vendor sensor binary is external evidence:

- SHA-256: `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`
- role/locator: `evidence/MANIFEST.tsv`

Firmware, dumps and other heavy primary evidence remain external and SHA-addressed.

## Next engineering phase

Prepare and reconcile the related OpenIPC implementation repositories against the camera contracts preserved here. Keep camera-level authority in this repository and move implementation changes into the repository that owns each component.
