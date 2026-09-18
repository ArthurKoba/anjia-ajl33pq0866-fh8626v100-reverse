# AJL33PQ0866 illumination, IR-cut and AUTO-light contract

This document promotes the closed stock reverse of the camera's illumination/day-night subsystem. Camera-level wiring and transition semantics are independent of Majestic vs Divinus.

## Physical board map

For ANJIA AJL33PQ0866:

- IR illumination: GPIO25, active value 1;
- white illumination: GPIO23, active value 1;
- IR-cut DAY actuator line: GPIO18;
- IR-cut NIGHT actuator line: GPIO60;
- stock configured IR-cut polarity/value: 0.

The illumination state machine is independent of WIDE/TELE lens selection on GPIO4/GPIO14.

## User scene states

Stock service control separates:

- forced DAY;
- forced NIGHT;
- AUTO;
- WLIGHT / white-light color mode.

WLIGHT is not the same state as NIGHT. Lens selection does not implicitly change day/night state.

## AUTO-light hysteresis

`REVERSE_CONFIRMED`: this exact board uses two LDR regions and a ten-sample moving history.

For the stock board configuration:

- DAY -> NIGHT when the smoothed metric is below 300;
- NIGHT -> DAY when the metric is above 500;
- 300..500 retains the previous state.

This is true hysteresis, not a single threshold.

Direction-dependent transition dwell defaults are 2000 ms for both DAY->NIGHT and NIGHT->DAY. Separate monotonic forbid timers prevent immediate retriggering; white-light operation has an independent longer forbid interval (stock default 600 in the recovered configuration surface).

## IR-cut actuator

`REVERSE_CONFIRMED`: the stock IR-cut controller owns GPIO18/GPIO60 as a two-line actuator, not as LED outputs.

- logical state 1 corresponds to DAY/visible-light filter position;
- logical state 0 corresponds to NIGHT/IR position;
- transition logic serializes the operation and honors board polarity;
- the actuator pulse/settle interval is approximately 190 ms;
- when `ircut_twice` is enabled, an optional second phase follows after approximately 200 ms.

A product implementation must keep the IR-cut actuator, IR illumination and white illumination as separate state variables.

## Illumination PWM

Stock product configuration gates PWM illumination behind `pwm_light`; default is `-1`, so automatic PWM brightness is disabled unless a board configuration explicitly enables it.

The recovered default PWM parameters are 20 kHz, eight levels and initial level 4. For level > 0:

- `period_ns = 1,000,000,000 / frequency`;
- `duty_ns = period_ns * level / level_count`.

Level 0 stops PWM.

`HARDWARE_PASS`: later target-live OpenIPC testing on this exact board proved an optional physical route from PWM0 to pad3 / IR GPIO25. A 2 Hz blink and a 20 kHz 0->100->0 percent duty sweep both drove the IR emitters, followed by safe restoration to GPIO25 low.

This proves the hardware capability; it does **not** change the stock product policy. Majestic/OpenIPC must not enable automatic PWM illumination without an explicit board policy and calibrated brightness curve.

## Stock PWM modes

The exact stock binary distinguishes several modes rather than one generic brightness flag:

- mode 0: ordinary GPIO illumination;
- mode 1: continuous automatic PWM brightness with threshold mapping and +/-1 level slew limit per update;
- mode 2: compiled-product special path whose exact helper is a stub in this Apollo build; do not invent homolog behavior;
- mode 3: PWM substitutes for IR GPIO switching;
- mode 4: PWM substitutes for white-light GPIO switching.

## DAY / NIGHT / WLIGHT policy

Recovered stock relationship:

- DAY: daylight processing, IR-cut DAY state, IR illumination off;
- NIGHT: night/monochrome processing, IR-cut NIGHT state, IR illumination according to board/control policy;
- WLIGHT: day-color processing/style, white illumination, IR off and its own low-light AE envelope.

Scene/profile transitions must be serialized with physical light/IR-cut changes. Do not collapse scene mode, illumination state, IR-cut state and ISP profile into one boolean.

## Safe shutdown boundary

`REVERSE_CONFIRMED` negative contract: stock graceful application teardown stops the light-control worker but does not necessarily force PWM off, IR off, white off or a defined IR-cut position. The synchronization destructor also does not stop physical outputs.

OpenIPC/Majestic should improve this behavior. Before final media-owner release or controlled shutdown:

1. stop/freeze AUTO scene transitions;
2. PWM off;
3. IR illumination off;
4. white illumination off;
5. set a board-defined safe IR-cut state, or explicitly preserve it by policy;
6. verify physical GPIO/PWM state;
7. then release synchronization/resources.

The safe IR-cut position is a board/product policy and must not be inferred merely from process exit.

## Builder implementation boundary

`SOURCE_CONFIRMED / HARDWARE REGRESSION PENDING`: Builder cleanup on 2026-09-18 keeps only the physical AJL33PQ0866 helper layer. The executable helper/init files now live in the ANJIA device-local board-support package shared by the composed Divinus/Majestic/diagnostic targets. The shared board overlay no longer owns AUTO/day/night/WLIGHT policy or any streamer state file. The helper handles GPIO25, GPIO23/SADC1 sharing and the GPIO18/GPIO60 actuator; a board init/shutdown service turns IR/white illumination off, restores pad70 to SADC and returns both IR-cut drive lines to rest without forcing a DAY/NIGHT filter move.

The current staged helper preserves its earlier electrical drive convention (`active=1`, `rest=0`) rather than silently changing behavior during source cleanup. The stock configuration fact recorded above (`ircut_polarity/value=0`) is a logical/configuration datum and is not, by itself, sufficient proof that the helper's electrical pulse levels should be inverted. The cleaned implementation therefore requires a physical DAY/NIGHT direction and active/rest regression before that drive convention is promoted to hardware acceptance. No new illumination target test was run for the current Builder composition tip during the latest documentation synchronization.

## Closure

Static reverse is implementation-ready for the supplied stock corpus: wiring/polarity, manual/AUTO control, hysteresis, dwell/forbid logic, PWM conversion/cache/modes, IR/white coordination, IR-cut timing and the shutdown negative contract are closed.

Any future work should be implementation/calibration/target acceptance rather than another broad illumination reverse.
