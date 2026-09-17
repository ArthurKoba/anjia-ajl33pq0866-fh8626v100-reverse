# Historical audio and network findings

This file preserves point-in-time audio/network observations. Current authority is under `docs/audio/` and `docs/hardware/`.

## RTX microphone and speaker hardware pass

`HARDWARE_PASS`: the OpenIPC RTX path established functional camera audio on AJL33PQ0866 through `/dev/rtxbus`, not ALSA/OSS and not the optional native `/dev/fh_audio` owner.

Microphone capture passed with mono 8 kHz signed 16-bit PCM and 320-byte packet configuration. A validated capture produced 64,000 bytes with non-zero signal, proving that the ARC producer/shared-memory DMA ring advanced after correcting the stock-proven audio clock masks.

RTX speaker/AO playback also passed. Stock-style playback used AO `type=3`, mono 16-bit, 320-byte packets and worked at both 8 kHz and 16 kHz. GPIO24 is the independent speaker-amplifier mute, active high.

The accepted AJL33PQ0866 ownership model keeps native `FH_DW_I2S`/`FH_DMAC` disabled while the RTX path owns codec/I2S/DMA hardware. This does not establish ALSA/OSS semantics, calibrated audio quality or that every stock audio feature uses the same API.

The current durable RTX/ARC contract is `../../docs/audio/rtx-arc.md`.

## Stock playback feature

Stock `/app` contained `/app/abin/playaudio` and WAV prompt assets under `/app/res/wav/`, which is positive evidence for an application-owned prompt playback feature.

`/dev/bgm` and `bgm.ko` are not audio. They belong to the FH8626 background-motion engine documented in `../../docs/media/bgm-motion.md`.

The presence of WAV files and `playaudio` alone does not recover the playback ABI.

## Historical RTSP ownership failure

One color/RTSP run was invalid because an older process owned TCP/18554 while the candidate failed to bind. It must not be cited as a successful candidate stream result.

Durable lesson: service acceptance requires candidate path/hash, PID/executable and listener ownership before interpreting stream behavior.

## Camera microphone/uplink observations

A 2026-08-28 stock capture sampled one scripted microphone/uplink cycle at five phases. Apollo remained the same process; sampled I2C0 and PWM counters stayed unchanged while ordinary media/network counters continued advancing.

The capture did not establish a microphone ABI, transport stream or ownership change. Aggregate network growth cannot be assigned to microphone traffic without socket/packet ownership evidence.

A second capture independently preserved the same limitation. The later RTX hardware pass proves one concrete camera-audio path but does not retroactively identify what the earlier stock toggle controlled.

## Stock siren observation

During one stock siren cycle, Apollo temporarily opened `/app/res/wav/beep.wav`. This directly supports an Apollo-owned WAV/prompt playback action in the sampled phase, but does not recover the playback ioctl/write ABI or calibrated speaker routing.
