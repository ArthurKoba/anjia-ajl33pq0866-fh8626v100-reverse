# FH8626V100 Builder cleanup handoff

Status: `SOURCE ARCHITECTURE COMPLETE / DOCUMENTATION SYNCHRONIZED / CI-BUILD-HARDWARE DEFERRED`.

Checked: 2026-09-18.

Builder is now a thin ANJIA AJL33PQ0866 device-composition layer. Generic FH8626 implementation remains in Linux/Firmware/Divinus; Majestic compatibility implementation remains in its Firmware direction.

No CI run, Builder/Firmware build or hardware test was requested or executed for the current Builder tip during this synchronization pass.

## Current refs and branch topology

Repository: `ArthurKoba/openipc-builder`.

Production/base:

`master@e0a643f4942b064a149f470b3c118ebba4daebb5`

Single active ANJIA development line:

`work/fh8626v100-anjia@9c507b85481df2787bb214e535f17003bdebb6e6`

There is no live separate Majestic Builder branch.

Preserved references:

- stock-style pre-cleanup PTZ controller: `archive/fh8626v100-anjia-stock-ptz-controller-20260918@a51eec5b294b03e8d16430e9018c3a0441647e49`;
- retired pre-composition Majestic Builder state: `archive/fh8626v100-anjia-majestic-branch-20260918`;
- old mixed Majestic experiment: `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`, with pre-experiment checkpoint `5603a701c8812aebc705c42e933ebae48aed805f`.

These are reference/provenance states, not active development branches.

## Runtime composition model

One physical ANJIA device tree exposes three selectable targets:

- `fh8626v100_lite_anjia-ajl33pq0866_divinus`;
- `fh8626v100_lite_anjia-ajl33pq0866_majestic`;
- `fh8626v100_lite_anjia-ajl33pq0866_diag`.

The generated Buildroot defconfig is composed as:

`Firmware generic fh8626v100_lite_defconfig -> ANJIA base.config -> target runtime fragment`.

The `firmware-base` file identifies the generic Firmware defconfig. Builder therefore does not carry a copied generic FH8626 full defconfig.

The ANJIA base contains only named-device deltas:

- board kernel fragment, including one-bit SD0 policy;
- removable FAT/microSD support policy;
- RTL8188FU selection;
- ANJIA board-support package.

Runtime fragments are intentionally short:

- Divinus selects Divinus plus the ANJIA Divinus config package;
- Majestic selects only the Firmware-owned FH8852V200 compatibility package;
- diagnostic selects no streamer and is intended for board/PTZ/lens/illumination/storage/network bring-up.

Each target has a matching `.firmware` metadata file. Default staging bindings are:

- Divinus -> `ArthurKoba/openipc-firmware.git@work/fh8626v100-divinus`;
- Majestic -> `ArthurKoba/openipc-firmware.git@work/fh8626v100-majestic`;
- diagnostic -> `ArthurKoba/openipc-firmware.git@work/fh8626v100`.

Explicit Firmware repo/ref environment overrides still take precedence for controlled bisect/debug work.

## Device-local package ownership

ANJIA packages live under:

`devices/fh8626v100_lite_anjia-ajl33pq0866/general/package/`

Current device-local packages are:

- `anjia-ajl33pq0866-board-support`;
- `anjia-ajl33pq0866-divinus-config`.

Builder registers these packages only for this device tree. They are no longer copied through root `package/`, so unrelated camera builds do not see ANJIA package definitions.

Board-support owns executable device hardware backends and board policy:

- low-level WIDE/TELE selector;
- illumination/IR-cut/SADC physical helper and safe-output init hook;
- persistent target/update identity service;
- optional PTZ motor backend.

Divinus YAML remains runtime-specific and is installed only when the Divinus variant selects its package.

## PTZ decision

Stock-like startup calibration/state management is not the production architecture.

Production PTZ is the optional Buildroot capability:

`BR2_PACKAGE_ANJIA_AJL33PQ0866_PTZ=y`

When selected, it installs the source-built `/dev/fh_pwm` relative backend and the standard OpenIPC entry point:

`gpio-motors PAN_STEPS TILT_STEPS DELAY_MS`

The low-level contract retains:

- accepted pan PWM11/10/9/6 and tilt PWM5/4/3/7 mapping;
- Fullhan PWM phase/enable/wait transaction shape;
- board timing/inversion inputs;
- single-owner lock;
- safe PWM disable and GPIO remux.

Production deliberately omits:

- startup calibration;
- PTZ movement merely because the camera booted;
- persistent inferred pan/tilt coordinates;
- absolute `goto` / `home`.

Current Divinus, Majestic and diagnostic variants explicitly enable PTZ, but future variants may omit it without losing lens, illumination or persistent device policy.

The previous stock-style controller remains reference/evidence only at the archive tag above.

## Lens / dual-sensor boundary

`fh8626-lens` remains independent from PTZ. It owns only the physical GPIO4/GPIO14 target-first selector and settle delay.

The selected media runtime still owns:

- stopping/restoring active VENC channels;
- mirror/flip preservation;
- exposure transient handling;
- GPIO5 cold-boot dual-sensor bootstrap around actual media-module/sensor startup.

Do not hide GPIO5 sequencing in a generic early/late Builder init service.

The current lens backend still uses the staged `/dev/gpiowave8` path. Replacement with a simpler GPIO/sysfs backend is a future hardware-evidence task; do not change that ownership path solely for cosmetic cleanup.

## Illumination

The board helper is runtime-neutral and owns physical operations only:

- IR LED GPIO25;
- white LED GPIO23;
- GPIO23/SADC1 shared-pad switching;
- IR-cut GPIO18/GPIO60;
- conservative safe output state at boot/shutdown.

AUTO, DAY/NIGHT/WLIGHT scene policy and ISP transitions remain media-runtime responsibilities.

The staged IR-cut helper still preserves its pre-cleanup electrical drive convention. Physical DAY/NIGHT direction and active/rest behavior remain a later hardware gate.

## Storage and persistent identity

Generic OpenIPC mdev remains removable-card hotplug/mount owner. ANJIA storage policy only prepares the recording directory for an already-mounted card and performs streamer-neutral sync/unmount at shutdown.

`fw_env.config` uses native OpenIPC `/dev/mtd1` 64 KiB environment geometry.

Runtime identity is now self-healing rather than relying on one-shot `custom.ok` behavior:

- `/etc/openipc/builder-target` records the exact composed target;
- `/etc/openipc/update-target` records the release asset stem, or is empty for diagnostic images;
- `S32anjia-env` validates/synchronizes persistent U-Boot policy on NOR boot and avoids rewriting unchanged variables;
- TFTP/initramfs boots do not mutate persistent environment;
- diagnostic images do not redirect the production upgrade target.

Per-device serial/cid/uuid/ethaddr remain untouched.

## Builder mechanics

Current work-line mechanics include:

- no self-`git pull`;
- exact checked-out Builder SHA is the build source;
- fork/non-default Firmware repo requires an explicit ref unless supplied by target metadata;
- Firmware checkout is prepared in a temporary path before replacing `openipc/`;
- branch/tag/SHA refs are resolved explicitly;
- checkout-level `flock` prevents concurrent local builds from sharing one mutable `openipc/`;
- conventional defconfigs and composed variant targets are both discovered;
- false `base.config` menu entries are excluded;
- device-local/global package name collisions are rejected;
- Builder-only composition metadata is removed from the Firmware tree before build;
- archive collection tolerates absent optional artifacts but rejects a successful build with no firmware artifacts;
- archives use second-resolution timestamps;
- when produced by the build, resolved Buildroot config and exact Builder/Firmware provenance are copied into the archive.

The GitHub App used by this project cannot write workflow files. An attempted `master.yml` dispatch polish received GitHub HTTP 403 `Resource not accessible by integration`. No other GitHub path was used. Leave workflow-file cleanup for an identity with the required permission.

## Validation status

Current tip `9c507b85481df2787bb214e535f17003bdebb6e6` is a **source architecture state**, not validated release state.

In this iteration, by owner direction:

- CI was not run;
- `.github/scripts/ci-matrix.py --self-test` was not run;
- no Divinus/Majestic/diagnostic Builder build was run;
- no physical-camera regression was run.

Earlier host/source checks validated the PTZ/lens backend before later package localization, optional-PTZ packaging and composed-variant changes. Keep those as historical source evidence only; they do not validate the complete current tip.

All three FH8626 composed targets therefore remain explicit `NOT_BUILT` entries.

## Deferred gates

When the owner chooses to enter validation:

1. run the Builder tree-wide CI selector self-test;
2. build all three exact composed targets and retain generated `build-info.txt`, input/resolved configs and image sizes;
3. hardware-check no PTZ boot movement, relative direction/speed/cancellation/safe stop;
4. hardware-check GPIO5 cold boot plus WIDE/TELE end-to-end media switching;
5. verify IR/white/IR-cut direction/rest behavior and GPIO23/SADC1 restoration;
6. verify microSD hotplug/shutdown, RTL8188FU, reset button and persistent env/update identity;
7. keep CI opt-outs until normal upstream Builder/Firmware ownership can resolve the required dependencies.

## Coordination rule

Builder changes are not considered handed off until this reverse repository is synchronized.

Whenever Builder changes its active SHA, branch topology, runtime target names, composition model, package ownership or hardware gates, update at minimum:

- `STATE.md`;
- `TASKS.md`;
- this handoff;
- any affected subsystem/process document.

This synchronization is part of the implementation workflow, not optional bookkeeping.
