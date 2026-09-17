# FH8626V100 camera components

This directory contains self-contained camera-specific components, contracts and tests that are useful inputs for integration into the OpenIPC repositories that own production code.

It is not a monolithic camera application and does not claim to be a complete build target.

## Contents

- `board/` — GPIO4/GPIO14 dual-sensor selector backend and rollback test.
- `control/` — generic algorithm host ABI and scene transaction policy with source-level test coverage.
- `diagnostics/` — bounded log-rotation helper and test.
- `media/` — camera timing policy plus the retained sidecar publisher reference primitive.
- `media_contracts/` — standalone media, codec, transport, lifecycle and observability contracts with host tests.
- `platform/` — ARM/uClibc-to-musl mmap compatibility shim for retained vendor libraries.
- `sensor/` — retained GC1054 profile inputs.

The sidecar publisher is reusable/reference transport code. Its presence here does **not** select a Majestic product transport architecture and does not require future integration to use the sidecar design.

## Host checks

Run the retained self-contained source checks with:

`make -C source/fh8626v100/components check`

The entry point covers board rollback policy, scene transaction policy, log rotation and the standalone media-contract test suite. It is host/source validation only and does not imply target hardware acceptance.

Working reverse reconstructions belong in the canonical Ghidra MCP project. Only source intended to be reused, integrated or directly tested belongs here.

Component source is evidence of implementation knowledge, not automatic target or product acceptance. Production changes belong in the repository that owns the relevant component.
