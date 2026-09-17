# Reverse analysis architecture

Status: `ACTIVE`.

## Canonical reverse workspace

The canonical mutable reverse-analysis environment for this camera is the Ghidra project exposed through Ghidra MCP and accessed through Koba MCP Bridge.

That project is the working index for functions, symbols, types, xrefs, strings, data, instructions, decompiler output and annotations used during focused reverse-engineering work.

## Git boundary

Promote durable results from Ghidra into Git when they are useful outside the analysis session:

- camera-level hardware or media contracts;
- concise function/address findings when the address remains useful;
- ABI/layout conclusions with an explicit evidence boundary;
- source-level reference implementations or tests intentionally retained;
- current integration implications.

Generated decompiler text, full disassembly dumps, xref tables and temporary analysis state are not Git authority.

## Evidence boundary

Primary or unique heavy inputs belong in the project evidence store and are referenced from `evidence/MANIFEST.tsv` by SHA-256 and locator. Examples include firmware, flash/rootfs/kernel/RAM dumps, retained vendor binaries and raw runtime/media captures.

Ghidra may consume those inputs, but generated analysis does not replace the primary bytes.

## Focused reverse workflow

1. Start from a concrete camera implementation or validation question.
2. Use the canonical Ghidra MCP project and target evidence.
3. Use neighboring Fullhan generations only as semantic hints when useful.
4. Prove FH8626-specific ABI/layout/register behavior from FH8626 target evidence.
5. Record the appropriate evidence class and confidence boundary.
6. Promote the durable result into the relevant current `docs/` or `source/` location.

Cross-Fullhan names, offsets and layouts are never target proof by themselves.

## Current boundary

Broad stock reverse of the exercised camera path is not a standing task. Existing reverse coverage is sufficient until a concrete implementation failure, hardware observation or contradiction requires focused analysis.
