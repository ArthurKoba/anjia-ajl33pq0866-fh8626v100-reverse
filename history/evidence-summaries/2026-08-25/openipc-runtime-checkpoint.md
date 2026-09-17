# OpenIPC MIPI/runtime checkpoint — 2026-08-25

Classification: retained direct runtime observation; historical context only, not current acceptance.

## MIPI-facing MMIO windows

The checkpoint captured point-in-time values from the FH8626 MIPI/ISP-facing windows at `0xF0000000`, `0xF0002000`, `0xF1000000` and `0xF1100000`.

### `0xF0000000`

Source SHA-256: `62ecb22108f4b4169177ca644ab3ca835691b842a5459e469972255f38b58f7d`

```text
+000 = 0x18112301
+004 = 0x00038607
+008 = 0x00000000
+00c = 0x07010071
+010 = 0x00010251
+014 = 0x00010132
+018 = 0x00000000
+01c = 0x00000400
+020 = 0x00000000
+024 = 0x0094020D
+028 = 0x00000174
+02c = 0x31000010
+030 = 0x11050505
+034 = 0x03010101
+038 = 0x00637863
+03c = 0x01070713
+040 = 0x00000000
+044 = 0x00000000
+048 = 0x01003310
```

### `0xF0002000`

Source SHA-256: `35dee7a7b859152ef77bbe14c310d1c67a70ad231ab66774f2641c5fd897aafe`

```text
+000 = 0x7940266B
+004 = 0x0000BFF8
+008 = 0x0F802020
+00c = 0x0000BFF8
+010 = 0x00000010
+014 = 0xEFBFFFF0
+018 = 0x00000000
+01c = 0x00000000
+020 = 0x00070000
+024 = 0xFFFFFFFF
+028 = 0x00000000
+02c = 0x00000000
+030 = 0x1B6C00C3
+034 = 0x1B6DB6DB
+038 = 0x90800C1A
+03c = 0x00200020
+040 = 0x00000001
```

### `0xF1000000`

Source SHA-256: `b5d3e3bf0c1e434e3b50704dfd443a04c367ddd99e024cfdf9e8500d9d5b7cc1`

```text
+000 = 0x00000000
+004 = 0x00000001
+008 = 0x00000001
```

### `0xF1100000`

Source SHA-256: `38bc0737144b9097e68fb1094f4f91aab37e95d69ed793917665a3352e336bcc`

```text
+000 = 0x3131312A
+004 = 0x00000000
+008 = 0x00000001
+00c = 0x00020006
+010 = 0x00000000
+014 = 0x00000000
+018 = 0x00000000
+01c = 0x00000000
+020 = 0x00000000
+024 = 0x00000000
+028 = 0x00000000
+02c = 0x00000000
+030 = 0x00000000
+034 = 0x00000000
+038 = 0x00000000
+03c = 0x00000000
+040 = 0x00000001
+044 = 0x00000001
+048 = 0x00030000
+04c = 0x00010001
+050 = 0x00000000
+054 = 0x00000A0A
```

These values corroborate active MIPI/ISP regions identified independently by target reverse. They are observations, not a universal initialization table.

## Same-boot context

Loaded Fullhan modules included `gpio_wave`, `bgm`, `jpeg`, `enc`, `isp`, `media_process`, `xbus_rpc` and `vmm`.

`/dev/mtdblock6` was mounted read-only as squashfs on `/mnt/stockapp`.

The observed VMM zone was `0xA2700000..0xA3F7FFFF`, 25088 KiB / 24.500 MiB, with no allocated blocks at capture time.

These snapshots describe one boot only and must not be generalized into fixed ownership/allocation rules.
