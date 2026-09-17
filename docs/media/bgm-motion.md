# FH8626 BGM background/motion engine

This document normalizes the exact 2026-08-31 reverse of stock `bgm.ko`. Evidence is `REVERSE_CONFIRMED` plus a bounded target query pass. It is not person-classification evidence and it is not an audio device.

## Role boundary

Stock firmware contains two separate detection paths:

```text
VPSS Y8 -> Apollo HDT/OBJDETECT -> person boxes
             ARM software classifier

VPU/VPSS -> bgm.ko -> background/foreground motion map
             FH8626 hardware + kernel post-processing
```

`/dev/bgm` belongs to the second path. The name BGM here means background model, not background music/audio.

## Kernel hardware contract

Stock `bgm.ko`:

- proprietary Fullhan module for Linux 4.9.129 / ARMv6;
- depends on `media_process`;
- driver/version strings `g45b1b95`, `V2.1.0.P5`, `2021-07-02`;
- registers misc device `/dev/bgm` (major 10, minor 50 on the exercised target);
- requires `bgm_hclk` and `bgm_clk`;
- maps MMIO `0xE8606000`, length `0x120`;
- uses hardware IRQ 7 through the Fullhan IRQ mapping layer;
- exposes ioctl magic `0x42` (`'B'`).

The driver is hybrid: hardware produces first-stage background/motion statistics, while kernel threads perform multi-Gaussian foreground detection, confidence/consistency processing, light/scene/still-image detection and background-model updates.

## Userspace ioctl boundary

Apollo uses `_IOWR` commands with magic `0x42`. Exact core commands recovered from target code include:

- `0xC0144200` / nr `0x00`, 20 bytes: memory initialization after allocation;
- `0xC00C4202` / nr `0x02`, 12 bytes: memory requirement query;
- `0xC0084203` / nr `0x03`, 8 bytes: set VI attributes;
- `0xC0084204` / nr `0x04`, 8 bytes: get VI attributes;
- `0xC0044205` / nr `0x05`: enable;
- `0xC0044206` / nr `0x06`: disable;
- `0xC0184207` / nr `0x07`, 24 bytes: submit frame;
- `0xC0304208` / nr `0x08`, 48 bytes: software/post-processed status;
- `0xC0984209` / nr `0x09`, 152 bytes: hardware status.

The recovered lifecycle is:

`query -> physically contiguous allocation -> memory init -> VI attr -> enable -> submit/bind frames -> results -> disable -> uninit`

The driver registers media object `0x11`. Advanced commands through nr `0x20` were mapped in the historical ABI report; raw field meanings that were not proved remain intentionally unnamed.

## Target query pass

The bounded target validation intentionally did not start the motion engine while another media owner was active.

`HARDWARE_PASS` / bounded ABI validation established:

- `/dev/bgm` existed as misc 10:50 mode 0660;
- memory query `0xC00C4202` was accepted;
- for current 1280x720 video the query returned a 160x90 grid and required 581,688 bytes;
- the VI-size getter reached the driver and returned vendor not-initialized status `0x800e4008`, proving the intended error path;
- engine execution/motion-map quality remained deliberately untested in that session because of the single-media-owner rule.

Therefore the correct status is: device identity and query ABI validated on target; full BGM engine execution still requires an isolated media-owner test if the product ever needs this path.

## Product boundary

BGM can supply low-cost foreground/motion regions. It does not classify people. A product human detector must use a separate classifier/backend; stock human detection is documented in `human-detection.md`.

Because `bgm.ko` combines hardware control with proprietary kernel-side post-processing, stock-module reuse, clean-room userspace adaptation and a future open driver are separate legal/engineering choices.
