# Repository map

This camera project coordinates several implementation repositories. Camera-level facts live here; component code changes live in the repository that owns the component.

| Component | Repository | Role |
|---|---|---|
| Camera project | `ArthurKoba/anjia-ajl33pq0866-fh8626v100-reverse` | Hardware/media contracts, target source, current state and evidence manifests |
| Builder | `ArthurKoba/openipc-builder` | Device profile and product-image assembly |
| Divinus | `ArthurKoba/openipc-divinus` | Open streamer/reference implementation |
| Firmware | `ArthurKoba/openipc-firmware` | OpenIPC firmware/platform packaging |
| Linux | `ArthurKoba/openipc-linux` | Kernel/platform changes |
| U-Boot | `ArthurKoba/u-boot-fullhan` | FH8626 bootloader work where required |
| Ghidra MCP infrastructure | `ArthurKoba/ghidra-mcp` | Reverse-analysis service deployment, not camera-level findings |

Known FH8626 engineering refs for the OpenIPC repositories are recorded in `docs/process/upstream-integration.md`.

## Authority boundary

Do not fork camera-level knowledge into implementation repositories. When implementation or hardware validation establishes a camera contract, document it here and reference it from the component change.

Ghidra MCP working state is not an implementation repository. Durable reverse conclusions return to this camera project; Ghidra service/deployment changes belong to `ghidra-mcp`.

## Branch policy

Every repository uses working branches for non-trivial changes. Agents do not create pull requests. Upstream-facing work must be curated from a verified upstream base into a coherent contribution series.
