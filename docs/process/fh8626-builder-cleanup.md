# FH8626V100 Builder cleanup handoff

Status: `READY FOR BUILDER REFACTOR / BUILD GATE PENDING`.

Checked: 2026-09-18.

The Builder repository can now be worked on independently without reopening the old mixed FH8626 Firmware experiment.

## Start here

Repository: `ArthurKoba/openipc-builder`.

Main ANJIA development line:

`work/fh8626v100-anjia@a51eec5b294b03e8d16430e9018c3a0441647e49`

Majestic-specific direction:

`work/fh8626v100-anjia-majestic@91314aa183e31070bb521364e81dacea569e08dd`

The Majestic direction already contains the latest shared ANJIA README/navigation update through an explicit merge from the main ANJIA line.

The obsolete historical Majestic experiment commit `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1` and its pre-Majestic parent `5603a701c8812aebc705c42e933ebae48aed805f` are provenance only. Their old live tags/branches were removed. Do not restore or build from that historical WIP.

## Ownership boundary

Builder is the named-camera assembly layer. It may own:

- ANJIA kernel fragment and board-only selections;
- RTL8188FU choice;
- removable-storage/device overlay policy;
- first-boot/customizer behavior;
- PTZ/lens/illumination device package and defaults;
- runtime selection for a named Builder target;
- device-specific excludes and configuration.

Builder must not re-acquire:

- generic FH8626 kernel config or Linux patches;
- generic FH8626 platform source;
- factory FH8626 `.ko/.so/.bin` payloads;
- Divinus source/patches;
- the Majestic FH8852 compatibility package itself.

Those are already separated into Linux/Firmware/Divinus or evidence ownership.

## Current useful Builder changes

The clean ANJIA line contains only current device staging plus one generic Builder feature:

- clean ANJIA device profile;
- source-built PTZ/lens board-support package;
- CI opt-out while cross-repository staging is fork-local;
- `builder.sh` support for `OPENIPC_FW_REPO` together with the existing `OPENIPC_FW_REV`.

The repository override is intentionally generic: without variables, Builder still clones upstream `OpenIPC/firmware`; with them, an FH8626 staging branch can be built without copying generic Firmware code into Builder.

## Known cleanup targets

These are good Builder tasks; they are not claims that each must be changed.

1. **builder.sh lifecycle.** Audit the self-`git pull`, destructive `rm -rf openipc`, error propagation, quoting/path assumptions, environment handling and archive behavior. Historical bring-up exposed a Buildroot failure from a dirty WSL `PATH`; decide whether Builder should sanitize or validate its build environment rather than relying on operator workarounds.
2. **Cross-repository staging UX.** `OPENIPC_FW_REPO` works locally, but CI/build-one plumbing currently centers on a Firmware ref. Decide whether repository override belongs in workflow inputs or should remain an explicit local/developer feature.
3. **Configuration duplication.** The named-device defconfig necessarily overlays Firmware, but it repeats toolchain/kernel/platform lines. Determine whether the duplication can be reduced without making Builder profiles opaque or dependent on fragile merge behavior.
4. **Runtime separation.** The main ANJIA profile is currently the Divinus development profile and includes `/etc/divinus.yaml`; the Majestic direction inherits the shared device tree even though it disables Divinus. Separate genuinely shared board configuration from streamer-specific configuration where doing so makes the assembled image clearer.
5. **Board-support package shape.** Review whether the current source-built PTZ/lens package and the large illumination shell helper have the right split, tests and init ownership. Preserve hardware contracts; simplify only where behavior stays explicit.
6. **Storage/automount policy.** Review `S39fh8626-storage`, `automount-hook`, excludes and SD lifecycle for duplication with generic OpenIPC behavior.
7. **Customizer and persistent config.** Verify that first-boot settings are idempotent and do not encode stale streamer-specific paths or old flash-layout assumptions.
8. **CI visibility.** Both FH8626 targets remain in `NOT_BUILT` because upstream Firmware does not yet contain the required core. Keep this explicit. Run `.github/scripts/ci-matrix.py --self-test` after Builder cleanup and remove the opt-out only when normal CI can actually resolve the Firmware dependency.
9. **Branch flow.** Shared ANJIA fixes go to `work/fh8626v100-anjia` and are then merged into `work/fh8626v100-anjia-majestic`. Majestic-only assembly changes stay on the latter. Do not create permanent micro-branches for routine edits.

## Do not conflate with current build acceptance

No new authoritative build was run after the branch cleanup and Majestic reconstruction.

A Builder refactor agent may perform static cleanup and lightweight source tests through repository/API tooling. The owner build/hardware gate remains required before any claim that the clean ANJIA or reconstructed Majestic target is accepted.
