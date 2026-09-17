# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot OpenIPC-native hardware acceptance and contribution curation

The agreed OpenIPC-native implementation is now the current U-Boot working line:

- `ArthurKoba/u-boot-fullhan/fh8626v100-mainline@227bcb40f68147864d778f1431973566cca383d8`
- equivalent implementation ref: `fh8626v100-openipc-native@227bcb40...`
- preserved hardware-proven factory-compatible recovery state: `fh8626v100-stock-compatible@49fe46e9ddb786e232d1359f9cee68c914a3a8db`

The source/build implementation pass is complete. Do not reopen stock-layout design work unless target evidence contradicts the recovered contract.

1. Build the final intended FH8626 OpenIPC `uImage` and measure it exactly. The target partition is the standard OpenIPC 2 MiB kernel partition.
2. If the kernel exceeds 2 MiB, first audit config, compression and built-in/module choices. Do not reintroduce the historical 3 MiB partition merely because it existed in a preservation snapshot.
3. Keep the implemented standard 8 MiB target layout: `256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.
4. Keep the board-specific internal boot split: reconstructed Fullhan Boot-ROM/DDR data at `0x00000` (64 KiB), U-Boot physical slot at `0x10000` (192 KiB), OpenIPC environment at `0x40000`.
5. Keep production on the implemented OpenIPC NOR environment contract: `kernaddr/kernsize`, `rootaddr/rootsize`, `mtdpartsnor8m`, `setnor8m`, `cmdnor`, `bootcmdnor`, `updatetool`, `ubnor/ubwrite`, `uknor/ukwrite`, `urnor/urwrite`. Factory compatibility stays RAM-recovery-only.
6. Use the board-qualified U-Boot updater `u-boot-fh8626v100-anjia-ajl33pq0866-nor.bin`. It must be exactly `0x50000` bytes: 256 KiB boot plus an erased 64 KiB environment sector.
7. Preserve the payload-derived native ROM descriptor implementation: actual raw U-Boot size, `0x100`-aligned ROM payload, calculated JAMCRC, flash offset `0x10000`, load/entry `0xa0800000`. Do not restore the factory fixed size/JAMCRC in the production path.
8. Perform the documented one-time migration only with a verified full-flash backup, serial console and external SPI programmer available.
9. Cold-boot the native layout and verify: ROM -> U-Boot from `0x10000`, compiled/default environment at `0x40000`, OpenIPC MTD map, kernel from `0x50000`, rootfs from `/dev/mtdblock3`, network/MAC propagation and normal userspace startup.
10. Exercise `fw_printenv`/`fw_setenv`, `setnor8m`, `ubnor`, `uknor`, `urnor` after the native cold boot before calling the update contract accepted.
11. Promote the native state to `HARDWARE_PASS` only after the complete migrated image passes those gates.
12. After hardware acceptance, rebuild/squash the iterative native implementation history into a clean upstream-facing contribution series. Curate after hardware testing so the exact tested tree remains traceable.
13. Run current Das U-Boot style/checkpatch contribution gates over the curated series.
14. Update `.github/workflows/build.yml` to native board-qualified artifact names using an identity with GitHub workflow-write permission. The current Bridge App cannot perform that mutation; do not work around the permission boundary and do not generate fake legacy artifacts for a green badge.
15. Keep the ANJIA board scope explicit. Do not label the artifact universal until another FH8626V100 board independently validates DDR/Boot-ROM/PHY/GPIO/RAM/flash compatibility.
16. Preserve and document the Boot ROM reconstruction provenance chain for OpenIPC maintainers.
17. Present OpenIPC maintainers with source, hardware evidence, provenance and artifact scheme and obtain the intended OpenIPC U-Boot repository ownership before moving source into the organization.
18. Once ownership/naming is agreed, integrate the accepted board boot artifact into normal OpenIPC Firmware image assembly.
19. Treat OpenIPC `defib` support as later recovery integration, not a blocker for source contribution.

Current source/build evidence at `227bcb40...`:

- 8 native artifact tests passed;
- production U-Boot compiled;
- raw U-Boot `0x2ef20`;
- aligned ROM payload `0x2f000`;
- calculated JAMCRC `0x0c6d419b`;
- 192 KiB physical slot leaves `0x1000` bytes after the aligned payload;
- generated `boot` artifact `0x40000`;
- generated `nor` updater artifact `0x50000` with erased environment sector;
- generated artifact self-inspection passed;
- RAM migration/recovery U-Boot compiled.

The existing GitHub workflow still ends red only because its final stock-era post-check looks for removed legacy artifact names/descriptor values. The self-validating `build.sh` stages passed before that obsolete check.

Detailed audit and migration contract: `docs/hardware/uboot-port.md`.

## P0 — kernel reconciliation

20. Audit `ArthurKoba/openipc-linux` branch `fullhan-fh8626v100` against the accepted platform contracts in this repository.
21. Locate and verify the exact upstream FH8626V100 kernel pull request, its current state, comments and any drift from the local branch.
22. Review board/kernel configuration and built-in/module choices. Keep open kernel/platform support distinct from proprietary Fullhan media modules.
23. Build and measure the final OpenIPC `uImage` as the immediate U-Boot/layout gate; avoid changing kernel functionality unless a defect, upstream review issue, regression or justified size/config cleanup is found.

## P0 — firmware ownership sanitation

24. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
25. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
26. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
27. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
28. Explicitly document proprietary `.ko/.so` dependencies that remain required and distinguish them from open platform support.
29. Drop the historical FH8626 3 MiB-kernel / `0x450000` rootfs assembly rule when the final measured kernel confirms the standard OpenIPC layout.

## P1 — Divinus target completion

30. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
31. Build the exact latest candidate reproducibly and record its identity.
32. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
33. Validate sensor/media startup, visible image, VENC and sustained RTSP.
34. Validate restart/reconnect and random-access/timestamp behavior.
35. Validate WIDE/TELE switching and board sensor bootstrap behavior.
36. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
37. Validate microphone and speaker/two-way audio where supported by the candidate.
38. Repair only failures reproduced on that latest target candidate.
39. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

40. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
41. Pin an exact Majestic candidate/build, libraries and canonical config.
42. Validate VI -> VENC -> sustained RTSP before ISP work.
43. Validate ISP/color/exposure/day-night after the base media path is stable.
44. Validate audio capture/playback/two-way behavior.
45. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
46. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

47. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
48. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

49. Use Builder last as the thin AJL33PQ0866 assembly layer.
50. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
51. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
52. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
53. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
