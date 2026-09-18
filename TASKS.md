# Current tasks

Only actionable current or next-phase work belongs here.

## P0 — U-Boot OpenIPC-native hardware acceptance and contribution curation

The agreed OpenIPC-native implementation is now the current U-Boot working line:

- `ArthurKoba/u-boot-fullhan/fh8626v100-mainline@7ac0aa7e83fb859b90c8b5e11bf617e367f2bd1d`
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
22. **Curated source series complete:** `work/fh8626v100@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`, 13 coherent commits over the verified base, with no audit metadata or fixup history.
23. The final tree contains the required FH8626 boardconfig, boardconfig-driven pin selection, standard OpenIPC MTD layout, RTC opt-in, neutral one-bit SD0 option, and legacy/AXI DMA registration behind their Kconfig symbols.
24. Preserve the hardware-tested FH8626 static RMII policy; JL1101, MAC and checksum changes are independently reviewable.
25. Do not carry the unrelated SADC string-copy cleanup or fork-local workflow/audit metadata into Linux.
26. Update the Firmware AJL kernel fragment to select `CONFIG_FH8626V100_SD0_1BIT=y` when Firmware is switched to the curated Linux series. Do not change that symbol alone while Firmware still applies its preserved old kernel patch set.
27. Locate/verify the exact upstream OpenIPC FH8626V100 pull request if accessible. Current inspection does not independently identify it; do not invent its number/status.
28. Owner-side validation gate: run the repository's contribution checks, build the exact OpenIPC image from `work/fh8626v100`, and record final `uImage` size. Historical hardware-accepted size was 1,583,456 bytes but does not validate the reconstructed tree.
29. Keep the standard 2 MiB kernel partition; if the final image exceeds it, audit config/compression/built-ins first.
30. Retest behavior-changing areas, especially the newly registered AXI-DMA path, before promoting the reconstruction beyond `SOURCE_CONFIRMED / AUDIT`.
31. Only after validation, perform the single owner-authorized update of PR-facing history if still required.
32. Agent work remains browser/API-first; no agent-side clone, Buildroot setup or heavyweight build unless explicitly requested by the owner.
33. **Production kernel-config source audit complete:** decisions and retained/dead symbols are recorded in `docs/process/fh8626-kernel-config-audit.md`; Firmware core candidate is now `work/fh8626v100@eabd1ccd...`; `c437d6eb...` remains the post-kernel-config-audit checkpoint before streamer neutralization.
34. Keep recovery-only NFSv3/IP-autoconfig/initrd and debugfs explicit during the current bring-up phase; do not remove them merely for size. Revisit a separate bring-up fragment only after the main hardware acceptance path is stable.
35. Owner build gate: resolve the exact final `.config` from core `eabd1ccd...`, record `uImage`/SquashFS sizes and hashes, and check whether Kconfig re-selected any requested-off symbol.
36. Hardware regression gate after that build: Ethernet, USB/RTL8188FU, microSD, SADC illumination, pinctrl/PTZ, watchdog, media/audio and normal NOR rootfs behavior.

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

24. Preserve `openipc-firmware/archive/fh8626v100-platform-20260918@f4bf49da...` as immutable provenance/evidence only.
25. Use `openipc-firmware/work/fh8626v100@80169887...` as the streamer-neutral core. It consumes Linux `work/fh8626v100@357c2d13...` and owns no AJL-specific board policy.
26. **Build ownership for the still-proprietary media kernel ABI is restored:** all FH8626 Firmware directions select `BR2_PACKAGE_FULLHAN_MEDIA_FH8626V100=y`.
27. The active source branches must contain no proprietary media bytes. `fullhan-media-fh8626v100` downloads the exact preserved `bgm/enc/gpio_wave/isp/jpeg/media_process/vmm/xbus_rpc` modules plus `rtthread_arc.bin` from immutable commit `f4bf49da...` and verifies every file with SHA-256. Independent Koba artifact copies are retained for provenance.
28. Keep the package limited to the unreplaced kernel/ARC runtime. Do not restore the archived vendor GC1054 plug-in, sensor profile objects or vendor `libmipi.so` into the current deployment path.
29. `load_fullhan -i` must integrate through OpenIPC `S70vendor` and preserve VMM -> XBUS/ARC -> media_process -> ISP -> enc -> JPEG -> BGM -> gpio_wave ordering. Treat missing media device nodes as boot/runtime failure, not a recoverable warning.
30. The retained modules currently report `vermagic=4.9.129 mod_unload ARMv6 p2v8`. Recheck actual module insertion on the final Linux build; static vermagic agreement is not hardware acceptance.
31. This package is **transitional deployment infrastructure**, not blob retirement. Continue the kernel/ARC replacement backlog in `docs/process/fh8626-blob-retirement.md`.
32. Replace the temporary ArthurKoba Linux tarball pin with an OpenIPC-owned ref only after the curated kernel series lands upstream.
33. Owner build gate: build the exact current core/Majestic direction and record final `uImage` and SquashFS sizes. Hard limits are 2048 KiB and 5120 KiB. The proprietary package adds about 798 KiB raw before SquashFS, so rootfs headroom must be measured, not assumed.
34. Hardware gate: prove module load, required device nodes, boot/MTD/rootfs_data and the selected media runtime before any claim of deployment readiness.

## P1 — Divinus target completion

35. Use `openipc-divinus/work/fh8626v100@6860cb9b...` as the current candidate and `openipc-firmware/work/fh8626v100-divinus@255b8c8d...` as its matching Firmware direction.
36. Keep Majestic/Divinus shared-contract coordination in reverse issue #3. The old issue #2 is closed as superseded.
37. Current Divinus already incorporates the critical Apollo/Ghidra lifecycle corrections: channel-valued VPU enable, distinct VPU disable, VI/VPU ownership split, StartRecvPic -> force-I -> media bind, and tightened H.264 RC/JPEG handling.
38. The Builder Divinus YAML intentionally keeps audio and JPEG/MJPEG disabled for the first target pass even though the implementation is broader. Do not confuse test sequencing with repository capability.
39. Run the focused host/source suite and then an exact ARM1176/musl build. Retain build identity, resolved config and image sizes.
40. Target-gate GC1054/ISP/media startup, sustained 1280x720@25 H.264, IDR/random access and reconnect behavior.
41. After base video passes, enable/test JPEG/MJPEG and RTX audio deliberately rather than silently widening the first acceptance run.
42. Validate graceful stop, same-boot restart/reconfigure epochs and client discontinuity behavior.
43. Validate WIDE/TELE/bootstrap/PTZ/illumination through their owning board layer; do not reintroduce generic Divinus GPIO policy.
44. Repair only failures reproduced on the exact built candidate; keep new shared ABI findings in issue #3.

## P2 — Majestic product path

45. **Offline selected-runtime closure is complete; build/hardware acceptance is not.** Current Firmware direction is `work/fh8626v100-majestic@527e3d8b...`.
46. The Majestic branch now includes source GC1054/MIPI, VMM, multi-channel H.264 SYS/VPSS/VENC/stream, full stock RC/readback/realtime controls, JPEG/MJPEG, motion YCmean/CPY, OSD GraphV2 and recovered RTX/ACW audio/VQE. H.265 is explicit unsupported because the FH8626 encoder stack has no HEVC engine.
47. The branch also selects the shared pinned `fullhan-media-fh8626v100` kernel/ARC runtime. Do not claim the userspace port works if the vendor media modules are absent.
48. Direct and transitive ABI guards must remain mandatory: a moving Majestic or selected donor library that reaches an unsupported SDK API must fail the build rather than boot with a fake-success stub.
49. Default boot remains media-off and native Majestic HTTP/WebUI remains untouched. No proxy or frontend JavaScript patch is accepted.
50. Owner build gate: run `./builder.sh fh8626v100_lite_anjia-ajl33pq0866_majestic`; record resolved Builder/Firmware/Linux SHAs, media-package SHA verification and final image sizes. Do not rely on the historical control-plane build as proof of the current tree.
51. Hardware ladder: media-off control plane -> media device nodes -> `majestic-fh8626-abi-probe` -> `majestic-fh8626-full-run`. Full-run must prove simultaneous main/sub RTSP, runtime RC/GOP/readback, JPEG, OSD, motion, audio input/output, speaker mute lifecycle and ANJIA day/night/IR-cut. Characterize `/metrics` at the real Majestic provider boundary; do not add another web server.

## P3 — firmware product integration

52. Keep `work/fh8626v100` as the shared core and keep the pinned proprietary media runtime package identical in core, Divinus and Majestic Firmware directions. Streamer-specific userspace must not leak into core.
53. Do not duplicate AJL camera policy, kernel patches or streamer source in Firmware. Generic SoC runtime packaging belongs in Firmware; board policy stays Builder; Linux source stays Linux.

## P4 — Builder cleanup and final device profile

54. Active Builder line is `ArthurKoba/openipc-builder/work/fh8626v100-anjia@9b9a7f0e...`. One ANJIA device tree composes `_divinus`, `_majestic` and `_diag`; the separate Majestic branch remains retired.
55. Preserve the composition rule: Firmware generic defconfig -> ANJIA `base.config` -> short runtime fragment. Do not reintroduce copied full defconfigs.
56. Builder contains no proprietary media binaries. The generic pinned media-kernel package is owned by Firmware and selected by the FH8626 base defconfig.
57. Keep ANJIA-only PTZ/lens/illumination/audio-mute/storage/update policy device-local. Production PTZ remains stateless relative movement; no boot calibration or inferred absolute coordinates.
58. Builder device README is the operator pre-deploy checklist and must stay synchronized with exact Firmware/Linux/U-Boot refs and hard size gates.
59. All three composed targets remain `NOT_BUILT` until owner builds record resolved configs and image sizes. For Majestic the immediate hard gates are `uImage <= 2048 KiB` and `rootfs.squashfs <= 5120 KiB`.
60. The GitHub App still cannot update workflow YAML and there is no usable current Actions run for these fork-local branches. Do not add workflow hacks; execute the owner build explicitly, then regress PTZ/lens/GPIO5/illumination/audio/SD/Wi-Fi/reset/persistent env on target.

## Standing repository rules

- Use a dedicated working branch for non-trivial changes in every repository.
- Verify the current upstream/base before editing a preserved FH8626 branch.
- Before implementation/contribution work, re-open the live upstream sources listed in `docs/process/openipc-upstream-rules.md`.
- Do not create pull requests as an agent.
- Keep camera-level facts and contracts in this repository.
- Keep active reverse work in canonical Ghidra MCP and use it only for concrete blockers.
- Keep heavy primary evidence in Google Drive and maintain current SHA-256/locator entries in `evidence/MANIFEST.tsv`.
- Never promote source/build/host evidence into hardware acceptance without target evidence.
