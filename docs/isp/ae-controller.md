# Stock GC1054 AE controller contract

This document promotes the durable implementation boundary recovered from the closed stock GC1054 AE reverse. Evidence class is `REVERSE_CONFIRMED` unless a stronger target class is cited elsewhere. It is not a claim that the current Majestic path already implements this controller.

The strongest retained reverse is the 2026-08-31 v5 closure, which validates the physical AE loop through the sensor-register boundary against real stock `day` and `night` runtime states and parses the complete `day/night/wlight` profile container.

## Control chain

Recovered stock chain:

`C4080 -> C26F0 -> C3F50 -> C949C -> C73F8 -> C757C -> C7894/C77DC -> C6D04 -> C90D4 -> C7C3C -> controller family -> C6AC8 -> GC1054 callbacks -> C9898 publication`

For the captured day state, `ctx+0x2F=0x80` is interpreted as signed `-128`, selecting `C7EB0 -> C883C` when the gate opens. An alternate controller family enters through `C7E04 -> C8134`.

`C9898` is publication/status after physical control work. It is not the exposure controller. `D0DF4/D0528/D0630` is a separate adaptive ISP block and does not call the GC1054 exposure/gain callbacks.

## Statistics/readiness clock

`C3F50` owns a statistics-readiness/cadence boundary:

- ioctl `0x00006933`; stop when not ready;
- ioctl `_IOR 0x8010690E`, 16 bytes;
- exact delay expression: `15000 + 1000*floor(1000/max(timing_word2,1))`;
- ISP phase state is toggled around that delay.

Do not substitute encoded-frame dequeue as the AE/AWB clock. Stock sensor/statistics cadence and public encoded cadence are distinct domains; see `../media/video-timing.md`.

## Measured value and target

`C73F8` publishes nine `u32` statistics into the main AE module at `+0x58..+0x78`.

For the captured day vector:

`[42,390,1128,40,405,1380,31,68,107]`

- sum = 3591;
- mean = 399;
- measured = `isqrt(mean << 12) = 1278`;
- with `ctx+0x30=80` and adaptive target disabled, target = `80 << 4 = 1280`;
- error = `-2`.

The night profile uses a distinct `C757C` statistics branch (`ctx+0x3D=1`) with centre-only reduction. The retained live night vector reproduces measured value `1547`, providing a second end-to-end stock numerical replay.

## 60-frame history and hysteresis

`C6D04` maintains the signed error history:

- `module+0xE0`: current error;
- `module+0xE4..+0x1D0`: 60 x `s32` history;
- `module+0x1D4`: aggregate.

Ordering is stock-specific: aggregate starts with the current error, adds the old 59 history words, shifts the history, stores the current error at `+0x1D0`, then stores the aggregate. Do not replace this with a generic post-shift moving sum.

`C7C3C` is the temporal/hysteresis gate. The retained day and night captures both have exact gate-state replays. A nonzero retained action word must not be interpreted as an action executed on the current frame when the gate returns zero.

## Profile parity and integration bounds

The complete 8648-byte `sensor_gc1054_mipi.bin` contains three exact 0xA58-byte profile payloads:

- `day` at offset `0x02C0`;
- `night` at offset `0x0D18`;
- `wlight` at offset `0x1770`.

The live alternate capture matches the static `night` payload in 2614/2648 bytes (98.716%). This identifies the runtime profile from context bytes rather than from scene darkness or lens assumptions.

Important AE-relevant profile differences include:

- day integration ceiling `ctx+0x38 = 899`;
- night/wlight integration ceiling `ctx+0x38 = 745`;
- day statistics branch `ctx+0x3D = 0`;
- night statistics branch `ctx+0x3D = 1`;
- wlight returns to branch 0;
- anti-flicker selector `ctx+0xA4C = 128` is unchanged across all three profiles.

`C6D70` applies the near-frame-limit guard against base VTS 899:

- day: `899 >= 895` -> effective ceiling `894`;
- night: `745 < 895` -> retain `745`;
- wlight: `745 < 895` -> retain `745`.

The historical night `max=745 / min=2` pair is closed by both static profile data and live runtime context. The captured day physical minimum is 1.

## Actuator families

For `C883C`, recovered action codes include:

- 0: no actuator action;
- 1: `C7058` direct integration correction;
- 4: gain correction/readback plus Q8/local ISP staging;
- 5: persistent Q8 auxiliary correction;
- 6: slow history feedback;
- 7: timing/integration synchronization;
- 9: `C72A0` anti-flicker quantized integration.

The alternate `C8134` family includes exposure-product redistribution through `C6E64`, quantized gain, local ISP staging, slow feedback and timing/integration handling.

`C7058` ultimately uses sensor callback `+0x10` (`set_intt`). `C6E64` preserves an exposure-product relationship, clamps/selects integration and stages residual exposure through gain.

## Deferred commit boundary

`C6AC8` consumes five `{value,dirty}` pairs:

| queue pair | consumer |
|---|---|
| `+0x00/+0x04` | sensor callback `+0x10` — `set_intt` |
| `+0x08/+0x0C` | sensor callback `+0x04` — `set_gain` |
| `+0x10/+0x14` | local `C58F8` ISP setter |
| `+0x18/+0x1C` | local `C598C` ISP setter |
| `+0x20/+0x24` | sensor callback `+0x14` — timing/frame setter |

`C58F8/C598C` operate on ISP `+0x168/+0x16C`; they are not hidden sensor callbacks. Dirty flags are cleared after commit.

The stock GC1054 callback table also exposes `get_gain` at `+0x0C`, `get_intt` at `+0x18` and raw `Sensor_Write` at `+0x3C`. `set_intt` writes registers `0x03/0x04` directly and does not perform a library-side clamp.

## Anti-flicker

`C72A0` is identified as `AE_SetSensorAntiFlick`. The selector comes from profile field index 27 / profile `+0x38`, captured as 128. The physical vendor mapping of value 128 to a named 50-Hz/60-Hz mode remains unproved and must not be guessed.

## Closure and remaining boundary

The stock physical AE/brightness-control scope is reverse-closed through concrete GC1054 registers for the recovered `day/night/wlight` profile families. Two independent real stock frames have numerical replays, and no requested physical-AE control-loop blocker remains in the historical reverse.

This closure does **not** mean a replacement owner already has exact stock runtime parity. Historical native-owner work still lacked exact statistics-epoch/barrier timing, full 60-frame history/gate behavior and five-pair `C6AC8` latency semantics at various points in its development.

For Majestic-first integration, do not implement this entire controller preemptively. Use it as the exact oracle only if current ISP/AE behavior becomes a concrete product blocker. If native owner work is resumed, preserve statistics readiness, profile-specific measurement, history/gate ordering, bounded actuator staging, deferred commit and generation invalidation rather than attaching AE to VENC dequeue.
