# Storage and authority architecture

This repository stays source/text-first and should remain useful without downloading large opaque artifacts or reconstructing an old workspace.

## GitHub

GitHub stores:

- current camera documentation and contracts;
- target-specific source and source-level tests owned by this project;
- compact current state/decisions;
- evidence manifests and provenance pointers;
- useful human-readable history.

Generic workspace tooling, generated reverse exports and mutable analysis databases do not belong here.

## Google Drive evidence store

Drive stores heavy or unique primary evidence:

- firmware and flash/rootfs/kernel/RAM images;
- retained stock/vendor binaries;
- raw UART/MMIO/runtime/media captures;
- other primary evidence that is inappropriate for Git.

Every external object used by current work should have a SHA-256, role/provenance and durable locator in `evidence/MANIFEST.tsv`.

## Ghidra MCP

Ghidra MCP is the canonical mutable reverse-analysis workspace.

Use it for function/type/xref/decompiler/data analysis and project annotations. Promote only durable conclusions or intentionally retained source-level references back to Git.

The Ghidra project database is working state, not camera documentation. Its deployment, host paths and service topology are infrastructure concerns and should not be duplicated in this repository.

## Primary vs generated evidence

Primary evidence and generated analysis are different classes. Never discard a unique primary input merely because Ghidra can regenerate a derivative view.

## Branching

Normal changes use working branches. Agents do not create pull requests. See `/AGENTS.md`.
