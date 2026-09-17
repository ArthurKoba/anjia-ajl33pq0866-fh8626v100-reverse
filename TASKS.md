# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot OpenIPC-native migration and contribution curation

The 2026-09-17 source/architecture audit of `ArthurKoba/u-boot-fullhan/fh8626v100-mainline` established a hardware-proven stock-compatible U-Boot baseline. That layout is a migration/reference state, not the intended final OpenIPC flash geometry.

1. Preserve the accepted stock-compatible branch/state as the known-good recovery reference.
2. Build and measure the final intended FH8626 OpenIPC `uImage`. Do not preserve the historical 3 MiB kernel reservation without evidence.
3. If the final `uImage` fits the current OpenIPC 2 MiB kernel partition, target the standard 8 MiB NOR layout: `256k(boot),64k(env),2048k(kernel),5120k(rootfs),-(rootfs_data)`.
4. For that target, restructure the first 320 KiB as: Fullhan Boot ROM container at `0x00000`, U-Boot 192 KiB slot at `0x10000`, OpenIPC environment at `0x40000`, kernel at `0x50000`.
5. Change the reconstructed Fullhan U-Boot descriptor to flash offset `0x10000` while retaining the validated RAM load/entry address, envelope/checksum contract unless hardware evidence requires otherwise.
6. Change U-Boot environment storage to `0x40000` and use an OpenIPC-native default environment rather than depending on the preserved vendor environment.
7. Remove stock-only runtime compatibility from the final product target where it is no longer required (`kload`, vendor GPIO syntax, stock bootfile/set_gpio policy); retain a migration/recovery configuration if useful.
8. Hardware-test the new first-320-KiB geometry from RAM/external recovery with a verified full-flash backup. The descriptor-offset/layout change is not `HARDWARE_PASS` until cold boot succeeds.
9. If the final kernel exceeds 2 MiB, first audit kernel config/compression/built-in choices and attempt to meet the standard layout. Use a SoC-specific partition map only if the required kernel genuinely cannot fit.
10. Make the ANJIA AJL33PQ0866 board-specific boundary explicit. Do not publish the current or new board artifact as universal FH8626V100 until another board independently passes DDR/bootstrap/PHY/GPIO/RAM/flash compatibility.
11. Curate the preserved FH8626 commits into a clean review series and fold the final `ethaddr` WIP change into its logical board/kernel-compatibility commit.
12. Keep generic DesignWare prerequisites separately reviewable from FH8626-specific Ethernet/SPI/board work.
13. Review the FH-specific vendor-GPIO compatibility change in generic `cmd/gpio.c`; it may remain in a stock-migration configuration but should not be required by the OpenIPC-native target.
14. Add explicit provenance documentation for the ANJIA Boot ROM container reconstruction and retained source evidence before external handoff.
15. Run current Das U-Boot contribution/style/checkpatch gates on the curated series in addition to existing build/bootchain CI.
16. Present source, hardware evidence, Boot ROM provenance and proposed artifact scheme to OpenIPC maintainers and obtain intended U-Boot repository ownership before moving source into the OpenIPC organization.
17. Once ownership/naming is agreed, integrate the accepted U-Boot artifact and matching layout into OpenIPC Firmware image assembly.
18. Treat OpenIPC `defib` support as useful later recovery integration, not a prerequisite for source contribution.

Detailed audit and migration design: `docs/hardware/uboot-port.md`.

## P0 — kernel reconciliation

19. Audit `ArthurKoba/openipc-linux` branch `fullhan-fh8626v100` against the accepted platform contracts in this repository.
20. Locate and verify the exact upstream FH8626V100 kernel pull request, its current state, comments and any drift from the local branch.
21. Review board/kernel configuration and built-in/module choices. Keep open kernel/platform support distinct from proprietary Fullhan media modules.
22. Measure the final OpenIPC `uImage` as part of the U-Boot/layout gate; avoid changing kernel functionality unless a defect, upstream review issue, regression or justified size/config cleanup is found.

## P0 — firmware ownership sanitation

23. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
24. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
25. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
26. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
27. Explicitly document proprietary `.ko/.so` dependencies that remain required and distinguish them from open platform support.
28. Replace the historical FH8626 3 MiB-kernel / `0x450000` rootfs assembly rule with the standard OpenIPC 8 MiB layout if the final measured kernel permits it.

## P1 — Divinus target completion

29. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
30. Build the exact latest candidate reproducibly and record its identity.
31. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
32. Validate sensor/media startup, visible image, VENC and sustained RTSP.
33. Validate restart/reconnect and random-access/timestamp behavior.
34. Validate WIDE/TELE switching and board sensor bootstrap behavior.
35. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
36. Validate microphone and speaker/two-way audio where supported by the candidate.
37. Repair only failures reproduced on that latest target candidate.
38. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

39. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
40. Pin an exact Majestic candidate/build, libraries and canonical config.
41. Validate VI -> VENC -> sustained RTSP before ISP work.
42. Validate ISP/color/exposure/day-night after the base media path is stable.
43. Validate audio capture/playback/two-way behavior.
44. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
45. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

46. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
47. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

48. Use Builder last as the thin AJL33PQ0866 assembly layer.
49. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
50. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
51. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
52. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
