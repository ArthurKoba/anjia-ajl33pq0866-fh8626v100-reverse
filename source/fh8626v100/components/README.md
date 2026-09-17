# FH8626V100 camera components

This directory contains self-contained camera-specific components, contracts and tests that are useful inputs for integration into the OpenIPC repositories that own production code.

It is not a monolithic camera application and does not claim to be a complete build target.

## Contents

- `board/` — GPIO4/GPIO14 dual-sensor selector backend and rollback test.
- `control/` — generic algorithm host ABI and scene transaction policy with source-level test coverage.
- `diagnostics/` — bounded log-rotation helper and test.
- `media/` — FH86 sidecar publisher and camera timing policy.
- `media_contracts/` — standalone media, codec, transport, lifecycle and observability contracts with host tests.
- `platform/` — ARM/uClibc-to-musl mmap compatibility shim for retained vendor libraries.
- `sensor/` — retained GC1054 profile inputs.

Working reverse reconstructions belong in the canonical Ghidra MCP project. Only source intended to be reused, integrated or directly tested belongs here.

Component source is evidence of implementation knowledge, not automatic target or product acceptance. Production changes belong in the repository that owns the relevant component.