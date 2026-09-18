# FH8852 Majestic sensor ABI -> FH8626 GC1054

Status: OFFLINE ABI CLOSED / BUILD AND TARGET PENDING

Checked: 2026-09-18.

This note records the sensor callback boundary used by the FH8626V100 Majestic
compatibility direction. It is based on direct Ghidra comparison of three
FH8852V200 donor sensor plug-ins, the preserved stock FH8626V100 GC1054 plug-in,
and the FH8852 libisp/libispcore consumers.

## Evidence

Dedicated Ghidra project:

- `fh8852_sensor_callback_audit`

Immutable artifacts imported for comparison:

- FH8852 GC4653: `sha256:1c40021fcdc8ec63c2a6c89c6b3f3baff40a017af9348a60caaa9b0b9be79916`
- FH8852 JXF32: `sha256:2f08764c1c4af8bd12a1f69d5b96049fa4dd1d2a4208aacf1a7dd571e157c9ee`
- FH8852 MN34425: `sha256:f57d563cc2bcd7852f84ae8a9648e91298a046ecae2dc29e87271f51fce85087`
- preserved stock FH8626 GC1054:
  `sha256:093c48710c9034dbde82681db014fd000a22f20487e74b6dbd147ebe4fff1eb9`
- FH8852 libisp:
  `sha256:aceca78aa85f463ff05f2cb68b993a966b6f9c75b58fb9f46d0aa3b920cfdb40`
- FH8852 libispcore:
  `sha256:3030b710a229e9ec017b3e0fe79e354d3cf39a1cec8203ad3d821f64d13083c5`

The stock FH8626 binary comes from preserved Firmware commit
`f4bf49da6ef355c9e733e00d774efe403513b1d4`.

## Table shape

FH8852 sensor plug-ins expose a 0x7c-byte ARM32 callback table.

FH8626 stock GC1054 exposes a different 0x68-byte callback table.

The FH8852 `libispcore.so` function `isp_core_sensor_install` copies exactly
0x7c bytes into its internal sensor table. Directly handing it the native
FH8626 0x68 object is therefore an ABI violation.

The Majestic compatibility package now exposes a typed 0x7c FH8852-shaped
facade. Every selected callback is non-NULL except the deliberately reserved
slot at +0x48. The target ABI probe checks the table at runtime without
initializing sensor hardware.

## Recovered FH8852 callback contract

| Offset | Callback | Selected GC1054 behavior |
| --- | --- | --- |
| 0x00 | name | `gc1054_mipi` |
| 0x04 | GetSensorViAttr | native FH8626 VI-attr translation |
| 0x08 | SetSensorFlipMirror | native source GC1054 mirror/flip |
| 0x0c | GetSensorFlipMirror | mutable output pointer, native readback |
| 0x10 | SetSensorIris | success/no-op, matching all three audited FH8852 donors |
| 0x14 | Sensor_Init | native source GC1054 init |
| 0x18 | SensorReset | no-op; physical GPIO5 bootstrap/reset stays board-owned |
| 0x1c | Sensor_DeInit | native I2C close |
| 0x20 | SetSensorFmt | native source format selection |
| 0x24 | Sensor_Kick | native callback, ENOSYS normalized to optional success |
| 0x28 | SetSensorReg | `uint16 reg, uint16 value` |
| 0x2c | SetExposureRatio | facade linear-mode state |
| 0x30 | GetExposureRatio | facade linear-mode state |
| 0x34 | GetSensorAttribute | exact `("WDR", out)` contract; GC1054 selected path is non-WDR |
| 0x38 | SetLaneNumMax | accepted but does not rewrite the hardware-proven fixed FH8626 GC1054 MIPI contract |
| 0x3c | GetSensorReg | `uint16 reg, uint16 *out`; not a direct register-value return |
| 0x40 | GetSensorAwbGain | facade state; native stock slot is absent |
| 0x44 | SetSensorAwbGain | facade state; native stock slot is absent |
| 0x48 | reserved | NULL |
| 0x4c | SensorCommonIf | returns -1 for this selected path, matching GC4653/JXF32 donors |
| 0x50 | GetAEDefault | reconstructed from GC1054 base frame length and max-integration margin |
| 0x54 | GetAEInfo | current integration/gain, base line-rate, current frame length |
| 0x58 | SetIntt | `value,index`; index 0 owns the selected linear exposure |
| 0x5c | CalcSnsValidIntt | mutable value pointer, clamped to current frame minus native margin |
| 0x60 | SetGain | `value,index`; index 0 owns the selected linear exposure |
| 0x64 | CalcSnsValidGain | mutable sensor-gain value; GC1054 source setter owns quantization |
| 0x68 | SetSnsFrameH | direct source GC1054 frame-length setter |
| 0x6c | GetMirrorFlipBayerFormat | native stock-equivalent Bayer-map pointer |
| 0x70 | GetUserSensorAwbGain | NULL; stock FH8626 GC1054 has no user preset table |
| 0x74 | GetSensorLtmCurve | NULL; stock FH8626 GC1054 returns no LTM curve |
| 0x78 | Sensor_Isconnect | self-contained I2C ID probe for 0x10/0x54 |

## AE semantics recovered from FH8852

Across the audited FH8852 donors:

- initial exposure ratio is 0x100;
- initial gain is 0x40;
- normal integration margin is 4 or 5 lines; the stock FH8626 GC1054 native
  callback proves 5 for this target;
- `GetSensorViAttr` records
  `line_rate = base_frame_length * nominal_fps`;
- `GetAEDefault.word4` is the base frame length used by
  `isp_core_set_sensor_frame_height`;
- `GetAEInfo.word3` is the current frame height;
- `CalcSnsValidIntt` and `CalcSnsValidGain` receive mutable pointers.

The selected GC1054 facade therefore keeps base frame length separate from the
current slow-shutter frame length. It does not reuse AE constants from GC4653,
JXF32 or MN34425.

## Important ABI correction: CommonIf

FH8852 `libispcore` invokes SensorCommonIf with integer commands 1, 2 and 3.

The native FH8626 GC1054 callback at +0x4c is a different API: it accepts
string queries such as `MAX_INTT_DIFF`, `STD_FRAME_RATE`,
`CUR_FRAME_RATE` and `REAL_FLIP_MIRROR`.

Those APIs must never be directly bridged by argument position. The earlier
facade did exactly that and was corrected. The FH8852 CommonIf path now follows
the donor behavior appropriate to this selected sensor path, while the native
string query remains an internal helper only.

## Lifecycle corrections

The source native GC1054 path now also guarantees:

- I2C fd is closed on partial initialization failure;
- `Sensor_Isconnect` can run before `Sensor_Init`, opening and closing I2C
  locally when needed;
- facade `Sensor_DeInit` closes native sensor state;
- facade `Sensor_Destroy` calls native destroy and resets facade-owned
  exposure/AWB state;
- the native dlopen handle is intentionally retained across same-process
  re-create, avoiding repeated dlopen references while preserving process-owned
  MIPI mapping bookkeeping.

## Current implementation checkpoint

Firmware Majestic implementation after this audit:

`ArthurKoba/openipc-firmware/work/fh8626v100-majestic@9160ef9e`

Later commits may advance the branch; use the branch head plus build provenance
for an actual test.

No target hardware acceptance is claimed by this document.
