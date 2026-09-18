# PTZ findings

## Closed hardware boundary

PTZ electrical/motor reverse is closed for the exercised hardware. Backend is `fh8626-ptz` over `/dev/fh_pwm`. After swapped motor connectors were corrected, stock-style type-2 movement was physically accepted on 2026-09-03.

Accepted channel mapping includes pan PWM11/10/9/6 and tilt PWM5/4/3/7.

Exact stock target capture observed ioctl request `0xC004700C` on the PWM path; LEFT payload began with channel 11 and UP with channel 5. PWM IRQ increased during movement while I2C counters did not, supporting the PWM motor boundary.

## Higher-level integration

Later HTTP/ONVIF adapter passed target curl movement/restore and preset lifecycles. Frigate static profile/FOV compatibility was implemented.

Remaining PTZ work is Frigate calibration/live person autotracking acceptance, not another motor-driver reverse.

## Methodological limit

The retained ioctl helper was narrowly filtered. Its successful capture is exact positive evidence for observed LEFT/UP events, not proof that no other stock control path exists. Exhaustive exclusivity is unnecessary unless a future blocker requires it.

## Historical directional IRQ corroboration

The 2026-08-29 capture set adds a same-session counter comparison around two stock PTZ directions. LEFT: before I2C0=5869, XBUS=15193, PWM=3141; after I2C0=5869, XBUS=16070, PWM=3355, giving deltas I2C0 +0, XBUS +877, PWM +214. UP: before I2C0=5869, XBUS=19930, PWM=3355; after I2C0=5869, XBUS=20272, PWM=3419, giving deltas I2C0 +0, XBUS +342, PWM +64. This independently corroborates PWM-backed motor activity with no I2C0 movement during the observed directional actions; it does not replace the stronger exact `0xC004700C` LEFT/UP ioctl payloads already retained.

Two historical breakpoint traces from the same acquisition family hit Apollo addresses `0x00204978` during a LEFT trace and `0x00207108` during a PTZ-start trace. Treat these as build-specific reverse landmarks only, not stable ABI/function addresses.


## OpenIPC production architecture decision

`SOURCE_CONFIRMED / HARDWARE REGRESSION PENDING`: Builder cleanup on 2026-09-18 deliberately stopped treating the stock-like startup calibration/controller as required product behavior.

Current Builder main ANJIA line is `ArthurKoba/openipc-builder/work/fh8626v100-anjia@9c507b85481df2787bb214e535f17003bdebb6e6`. Production exposes the existing hardware-proven PWM mapping through the standard relative OpenIPC entry point `gpio-motors PAN_STEPS TILT_STEPS DELAY_MS`. The motor backend is now an explicit optional Builder capability selected by `BR2_PACKAGE_ANJIA_AJL33PQ0866_PTZ=y`, so a future runtime variant can omit motors without duplicating or forking the rest of the ANJIA board layer.

The production backend keeps:

- `/dev/fh_pwm` and the accepted channel map;
- Fullhan PWM phase/enable/wait transaction shape;
- board timing/inversion inputs;
- a single-owner lock;
- safe PWM disable and GPIO remux after movement.

It deliberately removes:

- automatic startup calibration;
- any PTZ movement merely because the camera booted;
- persistent inferred position state;
- absolute `goto`/`home` policy.

There is no absolute encoder/position feedback in the accepted hardware contract, so a coordinate persisted across power loss is not authoritative physical position. If a future application needs absolute presets or autotracking calibration, that must be implemented as an explicitly validated higher-level policy rather than silently reintroduced into the motor backend.

The previous stock-style controller remains immutable reference/evidence at Builder tag `archive/fh8626v100-anjia-stock-ptz-controller-20260918@a51eec5b294b03e8d16430e9018c3a0441647e49`.

Earlier host recorder tests passed for the simplified motor transaction before the later device-local package/composed-variant/optional-capability refactors. No new tests or builds were run for the current Builder tip in the latest documentation pass. Physical direction, requested speed/delay, stop behavior and no-movement-at-boot still require target regression before the current packaged capability receives `HARDWARE_PASS`.
