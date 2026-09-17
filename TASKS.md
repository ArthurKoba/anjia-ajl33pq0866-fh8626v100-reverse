# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot OpenIPC-native hardware acceptance and contribution curation

The hardware-proven stock-compatible baseline remains `ArthurKoba/u-boot-fullhan/fh8626v100-mainline@49fe46e...`. The implemented OpenIPC-native candidate is `fh8626v100-openipc-native@99c47767...`. The candidate is five commits ahead, builds both production and RAM targets, and is not yet hardware-accepted.

1. Preserve `fh8626v100-mainline` unchanged as the known-good recovery/reference baseline until native cold boot passes.
2. Build the final intended FH8626 OpenIPC `uImage` and measure it exactly. The target partition is the standard OpenIPC 2 MiB kernel partition.
3. If the kernel exceeds 2 MiB, first audit config/compression/built-in/module choices. Do not reintroduce the historical 3 MiB partition merely because it existed in a preservation snapshot.
4. Use the implemented standard 8 MiB target layout: `256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.
5. Keep the board-specific internal boot split: reconstructed Fullhan Boot ROM data at `0x00000` (64 KiB), U-Boot at `0x10000` (192 KiB), OpenIPC environment at `0x40000`.
6. Use the native OpenIPC production environment and update commands already implemented on the candidate. Factory `kload`, vendor GPIO syntax and stock environment compatibility remain RAM-migration-only.
7. Perform the documented one-time migration with a verified complete flash backup, serial console and external SPI programmer available.
8. Cold-boot the native layout and verify: ROM -> U-Boot from `0x10000`, erased/default environment at `0x40000`, OpenIPC partition map, kernel from `0x50000`, rootfs from `/dev/mtdblock3`, network/MAC propagation and normal userspace startup.
9. Promote the candidate to `HARDWARE_PASS` only after the complete migrated image passes that cold-boot gate.
10. After hardware acceptance, curate/squash the five candidate commits into the final contribution series. The current extra executable-mode fix is implementation-history noise, not intended contribution structure.
11. Run current Das U-Boot style/checkpatch contribution gates over that curated series.
12. Update `.github/workflows/build.yml` to the native board-specific artifact names using an identity with GitHub workflow-write permission. The current bridge App cannot perform this mutation; do not work around the permission boundary.
13. Keep the ANJIA board scope explicit. Do not label the artifact `universal` until another FH8626V100 board independently validates DDR/Boot-ROM/PHY/GPIO/RAM/flash compatibility.
14. Document/retain the Boot ROM reconstruction provenance chain for OpenIPC maintainers.
15. Present OpenIPC maintainers with source, hardware evidence, provenance and artifact scheme and obtain the intended OpenIPC U-Boot repository ownership before moving source into the organization.
16. Once ownership/naming is agreed, integrate the accepted board boot artifact into normal OpenIPC Firmware image assembly.
17. Treat OpenIPC `defib` support as later recovery integration, not a blocker for source contribution.

Build evidence for the current candidate: production raw U-Boot `0x2ed18` bytes, fixed 192 KiB slot, 4836 bytes payload headroom after the four-byte ROM checksum fixup. TFTP upload, TFTP tuning variables, line editing, autocomplete, long help and `sleep` are enabled in the native production target.

The existing auto-workflow compiles production and RAM targets and generates the native artifacts, then fails only in obsolete stock-artifact post-build checks. This is an infrastructure/check definition mismatch, not a compile failure.

Detailed audit and migration contract: `docs/hardware/uboot-port.md`.

## P0 — kernel reconciliation

18. Audit `ArthurKoba/openipc-linux` branch `fullhan-fh8626v100` against the accepted platform contracts in this repository.
19. Locate and verify the exact upstream FH8626V100 kernel pull request, its current state, comments and any drift from the local branch.
20. Review board/kernel configuration and built-in/module choices. Keep open kernel/platform support distinct from proprietary Fullhan media modules.
21. Build and measure the final OpenIPC `uImage` as the immediate U-Boot/layout gate; avoid changing kernel functionality unless a defect, upstream review issue, regression or justified size/config cleanup is found.

## P0 — firmware ownership sanitation

22. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
23. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
24. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
25. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
26. Explicitly document proprietary `.ko/.so` dependencies that remain required and distinguish them from open platform support.
27. Drop the historical FH8626 3 MiB-kernel / `0x450000` rootfs assembly rule when the final measured kernel confirms the standard OpenIPC layout.

## P1 — Divinus target completion

28. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
29. Build the exact latest candidate reproducibly and record its identity.
30. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
31. Validate sensor/media startup, visible image, VENC and sustained RTSP.
32. Validate restart/reconnect and random-access/timestamp behavior.
33. Validate WIDE/TELE switching and board sensor bootstrap behavior.
34. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
35. Validate microphone and speaker/two-way audio where supported by the candidate.
36. Repair only failures reproduced on that latest target candidate.
37. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

38. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
39. Pin an exact Majestic candidate/build, libraries and canonical config.
40. Validate VI -> VENC -> sustained RTSP before ISP work.
41. Validate ISP/color/exposure/day-night after the base media path is stable.
42. Validate audio capture/playback/two-way behavior.
43. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
44. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

45. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
46. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

47. Use Builder last as the thin AJL33PQ0866 assembly layer.
48. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
49. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
50. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
51. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
