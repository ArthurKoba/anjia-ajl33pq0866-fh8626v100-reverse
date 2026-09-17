# FH8626V100 toolchain and userspace ABI boundary

This document records the durable toolchain/ABI conclusions recovered from exact-V100 external source/runtime research. It is an implementation constraint, not a claim that every vendor binary in circulation uses one identical SDK patch level.

## Exact family identity

`SOURCE_CONFIRMED / external exact-V100 evidence`:

- official Tuya indexing names the V100 toolchain family `arm-fullhanv2-linux-uclibcgnueabi-b2`;
- exact FH8626V100 AWS platform material targets uClibc / `arm-unknown-linux-uclibc`;
- exact-V100 runtime evidence reports GCC 5.5.0 with a 2019-era compiler marker.

Therefore FH8626V100 belongs to the older Fullhan-v2/uClibc userspace generation.

Do **not** treat newer Fullhan-v3 toolchains or FH8852V201 hard-float/uClibc binaries as drop-in V100 replacements. They are structural/reference material only unless a specific binary's ELF attributes prove compatibility.

## What the toolchain name does not prove

`gnueabi` alone does not establish hard-float vs soft-float calling convention. For any vendor binary/library that may cross into current implementation work, inspect the actual ELF and record at least:

- ELF class/endian/machine and EABI version;
- `Tag_CPU_arch`;
- `Tag_ABI_VFP_args` and FP architecture;
- interpreter;
- `DT_NEEDED` and symbol versions;
- exported/undefined symbols;
- build-id/version strings;
- SHA-256.

Do not infer these from the compiler prefix.

## libc boundary

Vendor uClibc objects must not leak libc-owned opaque state into a musl/OpenIPC process. In particular, do not pass across a runtime boundary:

- `FILE` / `DIR` objects;
- pthread/synchronization internals;
- locale objects;
- pointer-bearing vendor structures whose lifetime/packing belongs to the vendor runtime;
- compiler-dependent structures without an exact size/layout audit.

When proprietary/exact-uClibc ownership is unavoidable, the safe architecture is a process boundary with a typed versioned protocol using fixed-width fields. This is a generic ABI safety rule, not a requirement to revive the historical sidecar architecture.

## Integration consequence

The current Majestic-first path should prefer native/current OpenIPC interfaces and compatibility layers that avoid linking opaque V100 uClibc DSOs directly into the streamer process.

If an exact vendor helper is ever required for a concrete blocker, gate it on:

1. exact ELF identity/float ABI;
2. complete dependency closure;
3. typed boundary/layout audit;
4. restart/lifecycle isolation;
5. redistribution/license provenance.

The sensor/MIPI-specific uClibc↔musl boundary is documented in `../sensor/vendor-mipi-abi.md`.
