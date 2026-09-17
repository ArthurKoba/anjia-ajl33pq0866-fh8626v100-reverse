# Target identity — ANJIA AJL33PQ0866 / FH8626V100

This file keeps target-proven identity separate from OEM/homolog correlation. Homolog names must never replace target evidence.

## Canonical target identity

| Field | Value | Evidence |
|---|---|---|
| SoC | Fullhan `FH8626V100` | stock boot/runtime |
| Internal model | `AJL33PQ0866` | stock boot + flash |
| Stock firmware | `YGT.AJL33PQ0866-v230920.1051` | stock runtime |
| Fullhan platform SDK | `FH8626V100_IPC_V2.1.0_20210510` | stock boot |
| Fullhan platform patch | `FH8626V100_IPC_V2.1.0.FP19_20220221` | stock boot |
| App/IoT SDK numeric | `0x03051380` (`50664320`) | stock boot |
| Brand field | `ANJIA` | runtime + flash |
| Hardware ID | `AJ-SM-FH8626V100` | runtime + flash |
| Product environment | `PQEWN-01` | runtime + flash |
| Sensor family | dual `GC1054` MIPI | boot/runtime + hardware validation |
| Lens focal lengths | 3.6 mm + 12 mm | stock camera metadata + physical observation |
| Product default lens | WIDE / LensID 0 | stock config + hardware validation |
| PTZ | present | stock capability + `/dev/fh_pwm` reverse/validation |
| Microphone | present | stock capability + RTX hardware pass |
| Speaker/talkback | present | stock capability + RTX playback pass |
| IR | present | stock capability + illumination reverse |
| White light | present | stock capability + hardware evidence |
| Storage | microSD-capable | stock capability + SD0 hardware pass |
| SPI flash | Macronix `MX25L6405D`, 8 MiB | boot + canonical stock flash |
| Application ecosystem | CareCam Pro / CareCamPro | operator-observed UI; not required as board identity |

## PCB / assembly observations

Operator-confirmed physical tokens:

- `CF26`, `SM`, `V1.0`;
- PCB date `20210401`;
- approximate raw middle-string reading `CF26-54+545SM V1.0` — retain as approximate only;
- Ethernet magnetics `YYH-TEK YS1601HNL`;
- physical labels/connectors include `MIC`, `SPK`, `RST`, `VIN/GND/KEY`, UART `TX/RX`, motor connectors `左右` / `上下`, and `SPEED`;
- separate Wi-Fi module, PTZ mechanics and two optical paths are physically present.

## Network identity

Wi-Fi is target-confirmed RTL8188FU-class. Stock boot reports `dev_id=RTL8188FU`; a static supported-device list also contained other options, which must not be confused with the live selected device.

Ethernet is Fullhan `fh_gmac` on RMII. Stock boot reports PHY ID `0x937c4024` at MDIO address 0. External Linux material associates this ID with a JL1xxx Fast-Ethernet family, but the exact package remains homolog/reference until independently target-proven.

## Identity-layer rule

Do not collapse these layers into one public model name:

- internal model `AJL33PQ0866`;
- hardware ID `AJ-SM-FH8626V100`;
- brand `ANJIA`;
- product environment `PQEWN-01`;
- PCB CF26/SM tokens;
- retail/OEM family correlations.

Generic OpenIPC FH8626 support must not depend on the retail camera name. Camera/board integration may use target internal/PCB identity until a defensible public slug is selected.

## Evidence boundary

These identity facts are promoted from stock boot/runtime, flash evidence and physical target validation. Historical source paths are provenance only and do not define current authority.
