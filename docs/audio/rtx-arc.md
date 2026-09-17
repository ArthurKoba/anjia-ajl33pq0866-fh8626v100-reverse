# RTX / ARC audio contract

This document records the durable control/data boundary recovered from the FH8626 RTX audio path. It complements the physical acceptance summary in `findings.md`.

## Ownership path

The exercised stock/OpenIPC-compatible path is:

`FH_AC userspace -> ACW transport -> /dev/rtxbus -> xbus_rpc -> ARC RTThread audio service -> I2S/DMA ring -> codec hardware`.

This path is distinct from ALSA/OSS and from the optional native `FH_DW_I2S` / `FH_DMAC` Linux owner.

## Userspace and transport boundary

`REVERSE_CONFIRMED`:

- public/service calls include `FH_AC_Init`, `FH_AC_DeInit`, `FH_AC_Set_Config`, AI/AO enable/disable, AI volume and frame retrieval;
- ACW transports initialization, configuration, AI/AO enable/disable, frame traffic, AEC and capture/playback NR through `/dev/rtxbus`;
- recovered payloads include the command header, seven-word audio configuration, AI frame record, 64-bit PTS, NR parameters and the `0x182`-byte board-init payload;
- the AJL board bind policy returns `-1`, so generic ACW command 25 is not part of the exercised retail path;
- command 33 remains an alternate/full-duplex path without a justified public SDK symbol.

## Timeout failure mechanism

The historical capture failure was reconstructed as:

1. Apollo opens `/dev/rtxbus` and resets the transport;
2. `xbus_rpc:audio_open_clock` enables audio clocks through the Linux clock framework;
3. ARC accepts mono 8 kHz / 16-bit configuration, initializes the ring and accepts DMA start;
4. the AI frame path waits for the ring producer;
5. when DMA completion does not advance the producer, frame length is cleared and status `0x80130014` is returned.

Therefore `0x80130014` means no capture block was produced before timeout. It is not evidence that the command or format was rejected.

## Stock-proven audio clock masks

| clock | gate mask |
|---|---:|
| `ac_clk` | `0x00100000` |
| `acodec_mclk` | `0x00008000` |
| `acodec_pclk` | `0x00010000` |
| `acodec_pll_clk` | fixed 98.304 MHz |

Incorrect OpenIPC gate bitfields allowed control/RPC setup to succeed while the physical audio clocks remained gated, so the DMA producer never advanced. Correcting these masks converted the same RTX path into the later microphone hardware pass.

## Hardware acceptance

`HARDWARE_PASS` on AJL33PQ0866:

- microphone capture: mono, 8 kHz, signed 16-bit PCM, 320-byte packet configuration, 64,000 captured bytes with non-zero signal;
- speaker/AO playback: stock-style type 3, mono signed 16-bit, accepted at 8 kHz and 16 kHz with audible output;
- speaker amplifier mute: GPIO24 active high.

The accepted ownership model keeps native `FH_DW_I2S` / `FH_DMAC` disabled while RTX owns the codec/I2S/DMA path.

## Reverse authority

Working function names, types, xrefs, comments and further ARC/Apollo analysis belong in the canonical Ghidra project accessed through Koba MCP. Do not recreate local Ghidra databases, workspace-local script collections or generated decompiler-export trees in this repository.

Git retains only stable contracts such as those above. Heavy binaries and primary captures remain external evidence referenced through `evidence/MANIFEST.tsv`.

## Negative contracts

- Do not diagnose `0x80130014` as an unknown-command failure before checking DMA producer and clock state.
- Do not enable native DWI2S/DMAC simultaneously with RTX ownership on AJL33PQ0866.
- Do not infer ALSA/OSS semantics from the vendor transport.
- Do not add ACW command 25 to retail parity when the AJL bind policy is `-1`.
- `/dev/bgm` is unrelated to audio; it is the background-motion engine documented in `../media/bgm-motion.md`.

RTX/ARC ownership, transport shape, clock failure mechanism and the exercised physical input/output path are closed for this board. Reopen only for a concrete audio feature/quality blocker or contradictory target behavior.
