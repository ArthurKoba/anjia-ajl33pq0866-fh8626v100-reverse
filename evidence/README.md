# External evidence store

Heavy or unique primary evidence is stored in Google Drive rather than Git.

Project Drive root: `anjia-ajl33pq0866-fh8626v100-reverse` — ID `16l0gmS5RdYvgQlkUxczn2HYTu3y0U3vr`.

## Authority boundary

Git contains current source, documentation, contracts, state and the selected external-evidence manifest.

Drive contains primary bytes that are too large, opaque or inappropriate for Git, including firmware images, flash/rootfs/kernel/RAM dumps, vendor binaries, UART/runtime/media captures and other unique target evidence.

Ghidra MCP is the mutable reverse-analysis workspace. It may consume Drive evidence, but generated analysis is not primary evidence.

## Main Drive locations

- `evidence/` — ID `1rmuScnR-cGsEuTNPoYqBihexUdBegByb`
- `evidence/firmware-dumps/` — ID `1qS5OCwjYmMnCSJVUPhbqqxDXOSYzjrS3`
- `evidence/firmware-dumps/stock-firmware/` — ID `1cSC4bnc6gG4qSQ3Irqf0HAjlG8fxt-MG`
- `evidence/firmware-dumps/memory-dumps/` — ID `1BR6BNyMspLlLf-YpTDQKChsfd57034dM`
- `evidence/captures/` — ID `1kxRbef7XtCQZW3mhTb--dNVhwt8so-Xe`
- `evidence/reverse-inputs/` — ID `1Rr2LxVphv4pm2AqgScf44Nu8XIXTvAGx`

The retained GC1054 vendor sensor binary is stored in `evidence/firmware-dumps/stock-firmware/` and indexed in `MANIFEST.tsv` with SHA-256 `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`.

## Manifest rule

`MANIFEST.tsv` contains the selected external objects that remain useful to current engineering work. Each entry should have a SHA-256, a clear role and a durable Drive locator.

## Rules

- Keep one retained copy per unique primary object unless multiple physical copies are themselves meaningful evidence.
- Do not commit generated Ghidra databases, full disassembly exports, build caches or other bulky reproducible derivatives.
- Preserve unique primary inputs even when a derived analysis can be regenerated.
- Promote durable reverse conclusions into current documentation/source; keep mutable reverse working state in Ghidra MCP.
