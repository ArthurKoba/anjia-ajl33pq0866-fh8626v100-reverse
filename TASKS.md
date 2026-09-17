# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — repository authority

1. Keep this repository camera-level: hardware/media contracts, retained target source, evidence manifests, current state and concise engineering history.
2. Keep active reverse work in the canonical Ghidra MCP project and promote durable conclusions into Git.
3. Keep heavy primary evidence in Google Drive and maintain SHA-256/locator entries in `evidence/MANIFEST.tsv`.

## P0 — related repository integration

4. Reconcile the related OpenIPC repositories against the accepted camera contracts preserved here.
5. Curate retained implementation work into clean repository-owned changes in the component that owns each feature.
6. Keep camera-level contracts here rather than duplicating them across component repositories.

## P1 — Majestic product validation

7. Pin the exact Majestic candidate/build provenance, runtime libraries and canonical configuration.
8. Validate VI -> VENC -> sustained RTSP before treating the Majestic path as accepted.
9. Validate ISP/color/exposure/day-night after the base media path is stable.
10. Validate audio capture/playback/two-way behavior against explicit target criteria.
11. Keep `/dev/fh_pwm` as the accepted PTZ backend and continue with calibration/client/autotracking integration.

## Reference path

12. Keep Divinus useful as a diagnostic/reference implementation. Repair source-parity mismatches when they materially help validation or future integration.

## Reverse rule

Use Ghidra MCP for concrete blockers, contradictory evidence or focused implementation questions. Promote durable conclusions into the appropriate current subsystem document or source contract.
