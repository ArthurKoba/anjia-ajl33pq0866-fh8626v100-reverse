# Acceptance matrix

Never collapse source existence, host tests, successful ARM build, target execution and physical acceptance into one status.

| Area | Current status | Strongest evidence | Still open |
|---|---|---|---|
| Native OpenIPC/Linux boot on FH8626 | HARDWARE PASS for exercised platform bring-up | retained hardware/platform validation | release regression only after future changes |
| Ethernet / watchdog / core GPIO/I2C | HARDWARE PASS for recorded bring-up | retained Linux/platform validation | only regressions caused by later changes |
| GC1054 bootstrap / owner path | HARDWARE PASS for exercised owner/reference path | target validation + retained owner/source contracts | Majestic-specific VI compatibility |
| Exact owner v4.3 | SOURCE/BUILD ORACLE with hardware-proven lineage | retained owner v4.3 source | do not convert every owner feature into one monolithic PASS |
| Divinus external-owner transport baseline | TARGET STREAM PASS for the exercised historical candidate/run | retained target observations | reference path only; renewed implementation requires fresh acceptance |
| Divinus native FH8626 HAL migration | INCOMPLETE / REFERENCE | retained source + current Divinus documentation | parity repair only if reference work resumes |
| Divinus native source parity | CONFIRMED MISMATCHES at source level | owner/current/prior-working source comparison | repair/justify before renewed target acceptance |
| Board PTZ motor backend (`fh8626-ptz`) | HARDWARE PASS | retained hardware acceptance | release regression only |
| PTZ motor mapping | HARDWARE PASS after connector correction | retained PTZ evidence | preserve mapping and single-owner policy unless deliberately revalidated |
| Divinus HTTP/ONVIF PTZ adapter | TARGET CURL PASS for movement/restore/presets | retained ONVIF/PTZ integration evidence | Frigate calibration/live person autotracking |
| Builder AJL33PQ0866 profile | IMPLEMENTED; components hardware-backed | retained Builder/device-profile state | final image regression after current integration is rebuilt |
| Storage/recording profile | IMPLEMENTED / policy documented | retained device-profile state | product endurance/recovery acceptance |
| Majestic FH8852-family binary on FH8626 | OPERATOR-REPORTED PROCESS START | retained operator/project state | pin exact candidate and prove VI/VENC/sustained RTSP |
| Majestic ISP on FH8626 | NOT ACCEPTED | no complete current target acceptance | validate color/exposure/day-night after media path is stable |
| Majestic PTZ integration | ARCHITECTURE OPEN; motor backend accepted | `/dev/fh_pwm` hardware evidence | bridge control/API/ONVIF as required |
| Focused ISP semantic questions | ON-DEMAND REVERSE ONLY | canonical Ghidra MCP project + target evidence | reopen only for a concrete implementation/validation blocker |

## Promotion rule

A row is promoted only by evidence appropriate to its claim. Host tests never become hardware PASS by inference. Process start is not VI/VENC/RTSP proof. Source-level mismatch is not automatically a target failure; it is a reason to withhold acceptance until repaired or justified and tested. Independently hardware-accepted camera contracts remain accepted when the streamer changes unless contradictory target evidence appears.
