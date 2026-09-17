# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot audit/freeze

1. Audit `ArthurKoba/u-boot-fullhan` branch `fh8626v100-mainline` against the actual OpenIPC boot/recovery/update requirements.
2. Separate required production functionality from optional vendor/development commands and document intentional omissions.
3. Review generic-driver modifications, Kconfig/DTS/board split, configs, environment and contribution structure against current U-Boot/OpenIPC conventions.
4. Preserve the hardware-working port; change behavior only for a concrete defect, required missing capability or contribution-quality issue.

## P0 — kernel reconciliation

5. Audit `ArthurKoba/openipc-linux` branch `fullhan-fh8626v100` against the accepted platform contracts in this repository.
6. Locate and verify the exact upstream FH8626V100 kernel pull request, its current state, comments and any drift from the local branch.
7. Review board/kernel configuration and built-in/module choices. Keep open kernel/platform support distinct from proprietary Fullhan media modules.
8. Avoid new kernel development unless a defect, upstream review issue or regression is found.

## P0 — firmware ownership sanitation

9. Treat `openipc-firmware/fh8626v100-platform` as a preservation snapshot, not a final structure.
10. Inventory every FH8626 item in that snapshot and assign one owning repository before moving or deleting it.
11. Remove kernel patches from the future Firmware contribution set once the corresponding `openipc-linux` source is authoritative.
12. Move/curate one-camera behavior toward Builder and Divinus implementation toward Divinus; retain in Firmware only genuinely shared SoC/runtime integration that satisfies Firmware rules and provenance requirements.
13. Explicitly document proprietary `.ko/.so` dependencies that remain required and distinguish them from open platform support.

## P1 — Divinus target completion

14. Use `openipc-divinus/fh8626v100-canonical` as the latest source candidate, with the top WIP treated as source-only until retested.
15. Build the exact latest candidate reproducibly and record its identity.
16. Deploy it to the physical camera and prove candidate PID/executable/listener ownership before interpreting stream results.
17. Validate sensor/media startup, visible image, VENC and sustained RTSP.
18. Validate restart/reconnect and random-access/timestamp behavior.
19. Validate WIDE/TELE switching and board sensor bootstrap behavior.
20. Validate ISP/exposure/color/day-night behavior to the level actually exercised.
21. Validate microphone and speaker/two-way audio where supported by the candidate.
22. Repair only failures reproduced on that latest target candidate.
23. After hardware acceptance, curate a clean upstream-ready FH8626V100 Divinus series. The agent does not create the final pull request.

## P2 — Majestic product path

24. Keep the Builder Majestic experiment isolated until the Divinus reference path is closed.
25. Pin an exact Majestic candidate/build, libraries and canonical config.
26. Validate VI -> VENC -> sustained RTSP before ISP work.
27. Validate ISP/color/exposure/day-night after the base media path is stable.
28. Validate audio capture/playback/two-way behavior.
29. Integrate PTZ and illumination/IR-cut using the accepted camera contracts rather than rediscovering their low-level backends.
30. Reuse Divinus-derived knowledge only where it is architecture-neutral; do not assume source-level portability between streamers.

## P3 — firmware product integration

31. Once streamer ownership is settled, retain shared FH8626 SoC/runtime packages and load policy in Firmware where they genuinely belong.
32. Do not duplicate camera-specific profiles, kernel patches or streamer implementation source into Firmware.

## P4 — Builder final device profile

33. Use Builder last as the thin AJL33PQ0866 assembly layer.
34. Start later Builder work from the then-current upstream `master`, not by blindly extending the preserved diverged branch.
35. Use `5603a701c8812aebc705c42e933ebae48aed805f` as the preserved pre-Majestic reference checkpoint, not as a future upstream base.
36. Keep only per-device deltas: package selection, first-boot GPIO/bootstrap policy, sensor/lens defaults, camera-specific audio/PTZ/illumination config, excludes and other device-only packaging.
37. Do not retain duplicate kernel patches, generic FH8626 runtime code or Divinus/Majestic implementation source in the final Builder profile.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
