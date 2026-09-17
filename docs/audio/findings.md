# Audio findings

This page contains the current camera-level audio contract. Detailed historical experiments are retained under `history/audio/`.

## Accepted board path

`HARDWARE_PASS`: functional microphone capture and speaker playback were established on AJL33PQ0866 through the RTX path over `/dev/rtxbus`, not through a conventional ALSA/OSS device model.

Validated microphone contract:

- mono;
- 8 kHz;
- signed 16-bit PCM;
- 320-byte packet configuration.

A 64,000-byte capture showed a non-zero sample range and advancing ARC/shared-memory DMA state after the stock-proven audio clock masks were corrected.

Validated speaker/AO behavior:

- stock playback configuration used AO `type=3`;
- mono 16-bit;
- 320-byte packets;
- command/frame submission worked at 8 kHz and 16 kHz;
- physical playback was audibly reproduced.

GPIO24 is the independent speaker-amplifier mute, active high. Safe sequencing is to assert mute before configuration and restore mute before AO teardown.

The exact ARC ring/clock/ownership details are in `rtx-arc.md`.

## Ownership boundary

The accepted AJL33PQ0866 model keeps native `FH_DW_I2S`/`FH_DMAC` disabled while the RTX path owns codec/I2S/DMA hardware. Native DWI2S support remains generic FH8626V100 reference material rather than this board's production default.

Do not infer ALSA/OSS semantics from generic Linux expectations. Historical stock captures also did not expose conventional `/dev/dsp*`, `/dev/audio*` or `/dev/snd*` paths.

## Stock prompt playback

Stock firmware includes a `playaudio` utility and WAV prompt assets. This is positive evidence for stock prompt/WAV playback but does not define the userspace ABI used by the accepted RTX path.

`/dev/bgm` is unrelated to audio; it belongs to the background/foreground motion engine documented in `../media/bgm-motion.md`.

## Acceptance boundary

The exercised RTX microphone and speaker paths are hardware-proven. This does not by itself establish subjective audio quality, calibrated response, every stock audio feature or two-way product integration.

Resume audio work only when a concrete capture/playback/two-way integration requirement becomes active.

## Reverse boundary

If a new audio ownership/API question requires low-level analysis, use the canonical Ghidra MCP project. Durable conclusions return to this document or the relevant source contract.
