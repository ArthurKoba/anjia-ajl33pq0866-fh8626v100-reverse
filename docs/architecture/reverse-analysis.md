# Reverse analysis architecture

Status: `ACTIVE`.

## Canonical reverse workspace

The canonical mutable reverse-analysis environment for this camera is the Ghidra project exposed through Ghidra MCP and accessed through Koba MCP Bridge.

That project is the working index for functions, symbols, types, xrefs, strings, data, instructions, decompiler output and annotations used during focused reverse-engineering work.

## Opening the canonical headless projects

The Ghidra MCP backend may be healthy while no project is currently open. In
headless mode, an empty `list_projects()` call is not sufficient evidence that
the camera projects are unavailable: project discovery must search the mounted
project root explicitly.

Use this sequence through Koba MCP Bridge:

1. Call Ghidra `list_projects` with `searchDir=/projects`.
2. Select the concrete `.gpr` path for the subsystem being investigated.
3. Call `open_project` with that absolute project path.
4. Confirm `get_project_info` reports `has_project=true` and the expected
   `project_name`.
5. Discover program paths inside a headless project with
   `load_program_from_project(path="/does-not-exist", dry_run=true)` when
   necessary; its diagnostics list the available program paths. Do not treat
   `list_project_files` failure as missing data: that endpoint requires GUI
   mode in the current headless deployment.
6. Load the required program with `load_program_from_project` using its
   project-relative path, for example `/sensor/libgc1054_mipi.so`.
7. When more than one program is open, always pass the explicit `program`
   name to analysis tools.

Current mounted project root:

`/projects/anjia-ajl33pq0866-fh8626v100`

### GC1054 / MIPI userspace reverse

GC1054/MIPI userspace analysis belongs in the existing Apollo project:

`/projects/anjia-ajl33pq0866-fh8626v100/anjia_ajl33pq0866_fh8626v100_apollo.gpr`

Project name:

`anjia_ajl33pq0866_fh8626v100_apollo`

Programs currently present for this subsystem:

- `/sensor/libgc1054_mipi.so`
- `/sensor/libmipi.so`

The programs retain their full Ghidra analysis state, including function names
and comments, alongside `/apollo.unpacked`. The former standalone
`supplement_20260905/anjia_ajl33pq0866_fh8626v100_sensor_libs.gpr` was a
temporary duplicate analysis container and was deleted after its two programs
were migrated into the Apollo project. Do not recreate that standalone sensor
project.

These objects are exact retained FH8626V100 stock sensor/MIPI userspace
evidence. Reverse conclusions from them are FH8626 target evidence, subject to
the normal distinction between static reverse evidence and hardware
acceptance.

The mounted Ghidra workspace is the mutable analysis authority; Git records
the access procedure and durable conclusions.

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
