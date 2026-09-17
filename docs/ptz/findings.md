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
