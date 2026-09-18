# FH8626V100 Builder cleanup handoff

Status: `SOURCE CLEANUP COMPLETE / OWNER BUILD + HARDWARE GATES PENDING`.

Checked: 2026-09-18.

Builder has been reduced to the named ANJIA AJL33PQ0866 assembly/device layer. Generic FH8626 implementation was not moved back from Linux/Firmware/Divinus, and the obsolete mixed Majestic experiment was not restored.

## Current refs

Repository: `ArthurKoba/openipc-builder`.

Main ANJIA development line:

`work/fh8626v100-anjia@a39671f56267a403340354aebc82a3c889ac0df6`

Majestic direction after merging the shared cleanup:

`work/fh8626v100-anjia-majestic@19157b112a9ceeea25b7771bd79c2af8e3313558`

Preserved pre-cleanup PTZ reference:

`archive/fh8626v100-anjia-stock-ptz-controller-20260918@a51eec5b294b03e8d16430e9018c3a0441647e49`

The old mixed Majestic experiment `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1` and parent `5603a701c8812aebc705c42e933ebae48aed805f` remain provenance only.

## Ownership after cleanup

Builder owns:

- ANJIA board-only kernel fragment and one-bit SD0 selection;
- RTL8188FU selection;
- first-boot/persistent device policy without overwriting per-device identity;
- physical GPIO map and GPIO23/SADC1 shared-pad policy;
- low-level illumination/IR-cut helper and safe output handling;
- low-level WIDE/TELE selector;
- source-built FH8626 PWM motor backend adapted to the standard OpenIPC relative PTZ interface;
- removable-storage shutdown policy;
- named runtime selection and runtime-specific device configuration.

Builder does not own:

- generic FH8626 kernel config/source/patches;
- generic platform/media implementation;
- factory Fullhan `.ko/.so/.bin`;
- Divinus source/patches;
- the Majestic FH8852 compatibility package;
- product audio implementation.

## PTZ decision

The working stock-like controller was not accepted automatically as the product architecture.

Production Builder now exposes:

`gpio-motors PAN_STEPS TILT_STEPS DELAY_MS`

through a minimal `/dev/fh_pwm` backend. It retains the hardware-proven pan PWM11/10/9/6 and tilt PWM5/4/3/7 mapping, Fullhan PWM transaction/phase shape, pinmux handoff, owner lock and safe disable. A positive standard delay controls the nominal PWM period; delay 0 keeps the board defaults.

Removed from production:

- boot calibration;
- boot movement;
- persistent inferred pan/tilt coordinates;
- absolute `goto`/`home`;
- the assumption that a saved software coordinate remains physically true across power loss/manual movement.

The previous implementation remains at the archive tag above for evidence/debug/reference.

The lens selector is independent. `fh8626-lens` only drives GPIO4/GPIO14 in stock target-first order and waits for the physical selector. The selected streamer/media owner must coordinate VENC, mirror/flip and exposure around a logical lens switch.

GPIO5 is a separate cold-boot dual-sensor prerequisite. Its LOW/HIGH edges must surround the actual media-module/sensor startup and therefore must not be hidden in an unrelated Builder init script.

## Illumination decision

The board helper is runtime-neutral. It owns physical GPIO/pinmux operations only:

- IR LED GPIO25;
- white LED GPIO23;
- IR-cut GPIO18/GPIO60;
- SADC channel 1 on the shared GPIO23/pad70 path.

AUTO hysteresis, DAY/NIGHT/WLIGHT policy and ISP scene transitions are not implemented in Builder.

A board init/shutdown service places outputs in a conservative safe state: white/IR off, pad70 restored to SADC and both IR-cut drive lines returned to rest. It does not move the filter simply because the process starts/stops.

The current helper preserves the pre-cleanup staged IR-cut active/rest drive values. The reverse corpus records a stock polarity/value field of 0; those are not assumed to be the same semantic quantity. Physical DAY/NIGHT direction and electrical active/rest behavior are therefore an explicit hardware regression gate.

## Runtime separation

The shared device overlay no longer contains `/etc/divinus.yaml`.

Divinus selects a small `anjia-ajl33pq0866-divinus-config` package that installs the named-device YAML. The Majestic direction does not select Divinus and, after merge, inherits the same clean board layer without a dead Divinus configuration file.

The Majestic branch differs from main ANJIA only by:

- the Majestic-specific named defconfig;
- the additional Majestic target in CI `NOT_BUILT`.

Majestic implementation itself remains Firmware-owned.

## Storage/customizer

Generic OpenIPC mdev remains the removable-card hotplug/mount owner. The unused ANJIA automount hook was removed. The remaining storage init script only creates the recording directory for an already-mounted card and performs streamer-neutral sync/unmount at shutdown; it no longer calls a Divinus HTTP endpoint.

`fw_env.config` correctly addresses the native OpenIPC 64 KiB environment partition as `/dev/mtd1`. The first-boot customizer is NOR-SquashFS guarded, keeps serial/cid/uuid/ethaddr intact, sets the board-qualified update URL and selects RTL8188FU only when its module is present.

## Builder mechanics

The work line also fixes generic Builder lifecycle issues discovered during the audit:

- no self-`git pull` during a build;
- exact checked-out Builder revision is built;
- ambiguous device-defconfig lookup is rejected;
- whitespace-contaminated Buildroot `PATH` is rejected early;
- Firmware clone/copy/build/archive failures propagate;
- `OPENIPC_FW_REPO` + `OPENIPC_FW_REV` remains available for explicit staging;
- hi3518 autoupdate generation now happens before archive copy and its device condition is a normal shell test.

The repository override remains a local/staging feature. Normal CI should not be taught to fetch a contributor fork merely to make FH8626 staging appear upstream-ready.

## Source checks

Completed lightweight checks:

- FH8626 PTZ recorder tests: `PASS`;
- PTZ production source compiles with `-Wall -Wextra -Werror`;
- lens source compiles with `-Wall -Wextra -Werror`;
- changed shell files pass shell syntax checks;
- API compare confirms the Majestic direction is reconciled and contains only its intended two-file delta relative to main ANJIA.

Not completed by the API-only agent:

- literal `.github/scripts/ci-matrix.py --self-test` execution, because the project execution rules prohibit materializing/cloning the complete repository merely to run that tree-wide check;
- full Builder/Firmware build;
- target hardware regression.

## Remaining gates

1. Run `.github/scripts/ci-matrix.py --self-test` in a normal full checkout/CI context.
2. Build the Divinus named target against the matching Firmware direction and record resolved config plus image sizes.
3. Build the Majestic named target against `work/fh8626v100-majestic` and record the same.
4. On hardware verify no PTZ movement at boot, relative direction, standard delay/speed effect, cancellation and safe stop.
5. Verify GPIO5 cold-boot bootstrap and WIDE/TELE end-to-end runtime switching.
6. Verify IR/white LEDs, IR-cut DAY/NIGHT direction/polarity/rest drive, SADC shared-pad restore and safe shutdown.
7. Verify SD hotplug/shutdown, RTL8188FU, reset button and persistent environment/update policy.
8. Keep both FH8626 targets in `NOT_BUILT` until normal Builder CI can resolve their Firmware dependencies and the relevant gates pass.
