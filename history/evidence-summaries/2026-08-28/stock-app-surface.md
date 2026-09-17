# Stock application/device surface — 2026-08-28

This file preserves only durable target-surface observations from one stock session. Presence in the filesystem is evidence of deployment, not proof that every component was active in that session.

## `/app` inventory highlights

Source SHA-256: `564731acf28d1927e158c4e8f1b778692443d9e3a8545aa35c4a2f4274743734`

Relevant paths included:

- `/app/abin/apollo`
- `/app/abin/playaudio`
- `/app/bin/i2c_rw`
- `/app/bin/sensor_probe`
- `/app/lib/libgc1054_mipi.so`
- `/app/lib/libmipi.so`
- `/app/modules/bgm.ko`
- `/app/modules/enc.ko`
- `/app/modules/gpio_wave.ko`
- `/app/modules/isp.ko`
- `/app/modules/jpeg.ko`
- `/app/modules/media_process.ko`
- `/app/modules/vmm.ko`
- `/app/modules/xbus_rpc.ko`
- prompt WAV assets under `/app/res/wav/`
- `/app/sensor.def`
- `/app/userdata/fh410/gc1054_day.bin`
- `/app/userdata/ptz_initmove`

The stock image therefore carried the GC1054/MIPI sensor stack, Fullhan media modules, PTZ state and an application-owned prompt playback surface.

## Device surface

A retained `/dev` inventory included `/dev/bgm`, `/dev/fh_pwm`, `/dev/isp`, `/dev/pae`, `/dev/media_process`, `/dev/vmm_userdev`, three I2C nodes and `/dev/watchdog`.

Conventional `/dev/dsp*`, `/dev/audio*` and `/dev/snd*` paths were absent in the sampled session; this does not imply absence of vendor-specific audio capability.

## GPIO surface

The stock kernel exposed two GPIO chips:

- `gpiochip0`: base 0, 32 lines, label `FH_GPIO0`;
- `gpiochip32`: base 32, 32 lines, label `FH_GPIO1`.

This is direct target evidence for historical GPIO numbering above 31.

## Boundary

These observations describe the stock target surface. They do not replace subsystem contracts in current `docs/`, and filesystem presence alone does not establish runtime ownership or a specific ioctl/register sequence.
