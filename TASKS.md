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

20. **Static platform/history audit is substantially complete.** Current authority: `docs/process/fh8626-kernel-series-audit.md`.
21. Keep `fullhan-fh8626v100@0dfafa643770d78389e444c03f46f1711662eda6` read-only while cleanup proceeds.
22. **Curated source series complete:** `rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`, 13 coherent commits over the verified base, with no audit metadata or fixup history.
23. The final tree contains the required FH8626 boardconfig, boardconfig-driven pin selection, standard OpenIPC MTD layout, RTC opt-in, neutral one-bit SD0 option, and legacy/AXI DMA registration behind their Kconfig symbols.
24. Preserve the hardware-tested FH8626 static RMII policy; JL1101, MAC and checksum changes are independently reviewable.
25. Do not carry the unrelated SADC string-copy cleanup or fork-local workflow/audit metadata into Linux.
26. Update the Firmware AJL kernel fragment to select `CONFIG_FH8626V100_SD0_1BIT=y` when Firmware is switched to the curated Linux series. Do not change that symbol alone while Firmware still applies its preserved old kernel patch set.
27. Locate/verify the exact upstream OpenIPC FH8626V100 pull request if accessible. Current inspection does not independently identify it; do not invent its number/status.
28. Owner-side validation gate: run the repository's contribution checks, build the exact OpenIPC image from `rework/fh8626v100-final-series`, and record final `uImage` size. Historical hardware-accepted size was 1,583,456 bytes but does not validate the reconstructed tree.
29. Keep the standard 2 MiB kernel partition; if the final image exceeds it, audit config/compression/built-ins first.
30. Retest behavior-changing areas, especially the newly registered AXI-DMA path, before promoting the reconstruction beyond `SOURCE_CONFIRMED / AUDIT`.
31. Only after validation, perform the single owner-authorized update of PR-facing history if still required.
32. Agent work remains browser/API-first; no agent-side clone, Buildroot setup or heavyweight build unless explicitly requested by the owner.

## P1 — RTC / TSENSOR investigation (non-blocking)

This is an independent platform research task and does **not** block Firmware, Divinus or Majestic integration.

33. Determine whether AJL33PQ0866's FH8626V100 RTC/TSENSOR path is genuinely unavailable in hardware/board wiring or whether the required RTC/analog-domain initialization has not yet been reconstructed.
34. Preserve the current negative evidence boundary: both native and untouched stock Linux 4.9.129 time out through the same RTC command-core handshake, and the stock product configuration uses `hw_rtc=no`.
35. Do not interpret that parity as proof that the FH8626 RTC/TSENSOR IP itself is defective or absent.
36. Reconstruct the minimum stock initialization chain before attempting new writes: PMU/clock/reset/power-domain sequencing, RTC wrapper/core state, analog/TSENSOR configuration, efuse/calibration inputs and any stock userspace/kernel prerequisites.
37. Compare stock runtime register/state transitions with the current open driver and identify the first point where the expected RTC core idle/command handshake diverges.
38. Test TSENSOR only through evidence-preserving reads first. A valid result requires changing, physically plausible raw samples across temperature change; a static/default value is not acceptance.
39. Do not expose `rtc0`, thermal or hwmon temperature on AJL33PQ0866 until the corresponding path is demonstrated on hardware.
40. Do not perform speculative PMU/RTC writes merely to force the block alive. Any write experiment must be tied to a reconstructed stock sequence or an identified hardware contract.
41. If the block can be recovered, move the required generic SoC initialization into Linux and keep camera-specific enablement in the appropriate board/Firmware layer. If it cannot, retain RTC/TSENSOR disabled for AJL33PQ0866 and document the hardware-level reason.

## P0 — firmware ownership sanitation

24. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
25. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
26. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
27. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
28. Track proprietary media/runtime artifacts through `docs/process/fh8626-blob-retirement.md`. The preservation branch currently contains 16 binary artifacts: eight `.ko`, two `.so` and six `.bin` files. They are migration dependencies/evidence, not the desired final upstream architecture.
29. Do not delete a working blob merely to make the tree look clean. First identify its ABI, callers, hardware role, provenance and replacement boundary; then replace it with source-built code or a defensible SDK-derived artifact and hardware-test the replacement.
30. Prioritize contained replacements before reopening broad ISP reverse: `libgc1054_mipi.so`, `libmipi.so`, then separable kernel modules and generated sensor/profile data. Treat `rtthread_arc.bin` as a separate ARC/RTX firmware project.
31. The substantial open ISP/AE/AWB/CCM runtime already recovered in source reduces the remaining userspace algorithm work, but it does not by itself replace proprietary kernel media modules or ARC firmware.
32. Drop the historical FH8626 3 MiB-kernel / `0x450000` rootfs assembly rule when the final measured kernel confirms the standard OpenIPC layout.

## P1 — Divinus target completion

33. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
34. Build the exact latest candidate reproducibly and record its identity.
35. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
36. Validate sensor/media startup, visible image, VENC and sustained RTSP.
37. Validate restart/reconnect and random-access/timestamp behavior.
38. Validate WIDE/TELE switching and board sensor bootstrap behavior.
39. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
40. Validate microphone and speaker/two-way audio where supported by the candidate.
41. Repair only failures reproduced on that latest target candidate.
42. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

43. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
44. Pin an exact Majestic candidate/build, libraries and canonical config.
45. Validate VI -> VENC -> sustained RTSP before ISP work.
46. Validate ISP/color/exposure/day-night after the base media path is stable.
47. Validate audio capture/playback/two-way behavior.
48. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
49. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

50. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
51. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

52. Use Builder last as the thin AJL33PQ0866 assembly layer.
53. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
54. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
55. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
56. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
