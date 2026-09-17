# Project history

This directory preserves useful engineering chronology, evidence provenance and durable failure/dead-end lessons. It is not current project authority.

For present project state use `STATE.md`, `TASKS.md` and the current `docs/` tree.

## What belongs here

- consolidated engineering chronology;
- provenance for retained primary evidence and source decisions;
- historical observations that explain current contracts;
- failed approaches worth keeping so they are not repeated.

## Reverse-history boundary

Active reverse analysis lives in the canonical Ghidra MCP project.

Historical reverse provenance may remain here when it explains the origin of a durable finding or external evidence object. Current technical conclusions belong in the appropriate subsystem documentation.

## Evidence boundary

Heavy primary evidence remains in the Google Drive evidence store and selected current objects are addressed from Git through `evidence/MANIFEST.tsv` by SHA-256 and locator. Git history preserves human-readable provenance; primary bytes remain external.

A historical file may preserve a SHA-256 identity for an object that is no longer part of the selected current manifest. Such an identity is **provenance-only**: it documents what was used at that historical point, but it must not be treated as a currently retrievable evidence dependency unless a current `evidence/MANIFEST.tsv` row provides a durable locator.
