# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot contribution curation

The 2026-09-17 source/architecture audit of `ArthurKoba/u-boot-fullhan/fh8626v100-mainline` is complete. No missing capability was found for the currently exercised OpenIPC boot/recovery/update path.

1. Preserve the accepted hardware behavior and the `0x2bb00` Fullhan ROM-visible U-Boot envelope; the audited production build has only 820 bytes of headroom.
2. Make the current ANJIA AJL33PQ0866 board-specific boundary explicit. Do not publish the existing build as a universal FH8626V100 bootloader.
3. Curate the five preserved FH8626 commits into a clean review series and fold the final `ethaddr` WIP change into its logical board/kernel-compatibility commit.
4. Keep generic DesignWare prerequisites separately reviewable from FH8626-specific Ethernet/SPI/board work.
5. Review the FH-specific vendor-GPIO compatibility change in generic `cmd/gpio.c`; retain it pragmatically for OpenIPC if required, but avoid presenting it as ideal mainline structure without review.
6. Add explicit provenance documentation for the ANJIA Boot ROM container reconstruction and retained source evidence before external handoff.
7. Run current Das U-Boot contribution/style/checkpatch gates on the curated series in addition to the existing build/bootchain CI.
8. Present the source, hardware evidence and proposed artifact/layout scheme to OpenIPC maintainers and obtain the intended OpenIPC U-Boot repository ownership before moving source or publishing organization-level artifacts.
9. Once ownership/naming is agreed, add an FH8626-specific OpenIPC Firmware image-assembly path: kernel at `0x50000`, preserve `rootfs_data` at `0x350000`, rootfs at `0x450000`; do not reuse the ordinary `0x250000` rootfs offset.
10. Treat OpenIPC `defib` support as a useful later recovery integration, not as a prerequisite for accepting the U-Boot source.

Detailed audit: `docs/hardware/uboot-port.md`.

## P0 — kernel reconciliation

11. Audit `ArthurKoba/openipc-linux` branch `fullhan-fh8626v100` against the accepted platform contracts in this repository.
12. Locate and verify the exact upstream FH8626V100 kernel pull request, its current state, comments and any drift from the local branch.
13. Review board/kernel configuration and built-in/module choices. Keep open kernel/platform support distinct from proprietary Fullhan media modules.
14. Avoid new kernel development unless a defect, upstream review issue or regression is found.

## P0 — firmware ownership sanitation

15. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
16. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
17. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
18. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
19. Explicitly document proprietary `.ko/.so` dependencies that remain required and distinguish them from open platform support.

## P1 — Divinus target completion

20. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
21. Build the exact latest candidate reproducibly and record its identity.
22. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
23. Validate sensor/media startup, visible image, VENC and sustained RTSP.
24. Validate restart/reconnect and random-access/timestamp behavior.
25. Validate WIDE/TELE switching and board sensor bootstrap behavior.
26. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
27. Validate microphone and speaker/two-way audio where supported by the candidate.
28. Repair only failures reproduced on that latest target candidate.
29. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

30. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
31. Pin an exact Majestic candidate/build, libraries and canonical config.
32. Validate VI -> VENC -> sustained RTSP before ISP work.
33. Validate ISP/color/exposure/day-night after the base media path is stable.
34. Validate audio capture/playback/two-way behavior.
35. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
36. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

37. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
38. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

39. Use Builder last as the thin AJL33PQ0866 assembly layer.
40. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
41. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
42. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
43. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
