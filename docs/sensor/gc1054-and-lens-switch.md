# GC1054 and dual-lens switching

## Hardware

- Dual GC1054 MIPI sensors.
- Product default lens: WIDE.
- Stock exposes WIDE/TELE logical lenses.
- Lens metadata: WIDE 3.6 mm, TELE 12 mm.

## Stock sensor boundary

`REVERSE_CONFIRMED`: stock runtime maps `libgc1054_mipi.so` and `libmipi.so`; sensor access goes through the vendor sensor/I2C wrapper. Historical exact reverse identified ioctl values `0x704`, `0x706`, and `0x707` in this path.

Do not treat direct callback invocation as automatically equivalent to the higher-level stock sensor wrapper; historical owner work missed wrapper side effects/order and regressed startup behavior.

## Cold-boot prerequisite

`HARDWARE_PASS`: dual-sensor visibility requires one board bootstrap after cold boot:

1. common sensor reset GPIO5 LOW;
2. load the validated Fullhan media module sequence;
3. GPIO5 HIGH;
4. only then prepare both GC1054 targets.

Without this bootstrap TELE was not physically visible even to stock `sensor_probe` despite correct GPIO4/GPIO14 selector state. GPIO5 belongs in board/bootstrap initialization, not every runtime lens switch.

## One-time startup preparation

Historical reverse/runtime validation established this stock-style preparation pattern:

1. select WIDE/target1;
2. wait about 600 ms;
3. initialize sensor and set valid format;
4. select TELE/target2;
5. wait about 600 ms;
6. initialize sensor and set the same selected format;
7. preserve mirror/flip state;
8. leave WIDE selected as product default.

Do not recreate the ISP/sensor stack during each normal runtime switch.

## Runtime lens selector

`STOCK_RUNTIME`: direct stock UART evidence confirmed captured selector order:

- logical lens 1 -> sensor 2 -> GPIO14 asserted before GPIO4 deasserted;
- logical lens 0 -> sensor 1 -> GPIO4 asserted before GPIO14 deasserted.

Endpoint mapping is WIDE GPIO4=1/GPIO14=0 and TELE GPIO4=0/GPIO14=1 for the captured stock firmware/session.

The runtime transaction is lightweight and separate from sensor-stack creation:

1. resolve target and no-op if already active;
2. serialize the switch path;
3. stop receive only on VENC channels that were active;
4. save mirror/flip when enabled;
5. switch GPIO4/GPIO14 target;
6. wait about 60 ms;
7. restore mirror/flip;
8. allow an outer settle of about 50 ms;
9. restart only channels that were active before the switch;
10. set sensor integration and gain to the exact transient reset values 64/64;
11. release the switch lock;
12. update focal metadata only at the outer API/application layer if needed.

The exact timings and channel-record count are historical stock implementation details and should not be generalized to unrelated firmware without validation. The durable contract is the ownership/order boundary: stop active encoding around the physical selector, preserve orientation, do not rebuild the media stack, and restore the previous active-channel set.

## State-machine boundaries

A pure lens switch must not:

- reload day/night or white-light profiles;
- reconstruct/reset the AE module/history;
- call Sensor_Create/SensorInit/SetSensorFmt per switch;
- stop audio;
- reconfigure MIPI;
- implicitly toggle IR, white light or IR-cut.

Lens target, day/night/illumination, IR-cut and audio are independent service planes.

Controlled stock validation proved WIDE -> TELE -> WIDE with one shared AE context/history. TELE reached the day integration ceiling and raised gain; returning to WIDE reconverged near the prior exposure. The 64/64 reset is an exact static contract and should be treated as transient rather than the desired steady exposure state.

## Rejected TELE diagnostics

Historical failed experiments established several negative rules that should not be retried as root-cause claims:

- direct `Sensor_Read(F0/F1)` returning `FFFF` is **not** a physical-presence discriminator; the same guessed path returned invalid values on working WIDE;
- restoring the observed `/dev/isp` ioctl `0x40046931` by itself did **not** restore TELE visibility;
- low-first vs high-first GPIO4/GPIO14 ordering alone was insufficient once the exact stock selector order was restored;
- missing GPIO4/GPIO14 pinmux and a hypothetical hidden sensor-selection API were also insufficient explanations.

For hardware visibility, use the validated product environment/bootstrap and stock-style sensor probing. The decisive missing prerequisite in the historical OpenIPC failure was the common GPIO5 cold-boot reset/bootstrap sequence above.

## ISP prerequisite lead

`REVERSE_CONFIRMED / not HARDWARE_PASS`: historical wrapper reverse shows a `/dev/isp` ioctl `0x40046931` occurring before sensor callback initialization. This remains an implementation lead until a clean target run proves the exact argument/ordering contract. It must not be promoted to the already-rejected claim that this ioctl alone explains TELE visibility.

## Reverse policy

Large GC1054/MIPI disassembly packs and raw captures remain external evidence. Promote only stable contracts and targeted findings here. Broad lens-switch reverse is closed unless contradictory target evidence appears.
