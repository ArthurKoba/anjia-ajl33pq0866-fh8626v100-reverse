# AE integration boundary

This page defines the current AE integration boundary. The exact stock GC1054 day/night/wlight controller oracle is `ae-controller.md`; that document is the current implementation-facing authority for recovered stock AE sequencing and arithmetic.

## Current authority

- `ae-controller.md` contains the durable `REVERSE_CONFIRMED` stock controller contract.
- Majestic is the preferred product path; current work must not recreate a native stock AE controller unless a concrete integration blocker requires it.
- Media acceptance remains ordered: establish reproducible VI -> VENC -> sustained RTSP first, then validate exposure/color/day-night behavior.
- Source/reverse parity is not target acceptance. AE acceptance requires target behavior under explicit lighting conditions and known sensor/profile state.

## Historical engineering lessons

Historical native-owner work exposed two useful failure classes:

1. Lens switching can leave transient exposure/gain state behind when no automatic statistics provider restores the revisited lens state. Historical replacement code cached per-lens integration/gain and restored it as an owner-side recovery policy; this is not an inferred stock atomic-switch contract.
2. One temporary owner candidate improved brightness and removed a green-highlight failure in an operator observation while white clipping remained. That observation was scene-specific and did not establish controlled AE acceptance.

The old Apollo AE runtime slice and AEV1 stripe captures are retained only as historical provenance, not as dependencies of this current authority page. Their recorded identities and limitations are preserved in `../../history/isp/ae-engineering-history.md`. An object that is not indexed in `evidence/MANIFEST.tsv` must not be treated as a currently retrievable evidence dependency.

## Reopen rule

Re-enter focused AE reverse through the canonical Ghidra MCP project only when a current implementation or target-validation question contradicts or exceeds `ae-controller.md`. Promote any new durable conclusion back into the current ISP contract; keep temporary decompiler/export state out of Git.
