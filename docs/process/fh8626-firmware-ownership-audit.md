# FH8626V100 Firmware ownership and provenance audit

Status: `ACTIVE / CLEAN_FIRMWARE_CANDIDATE`.

Checked: 2026-09-18.

This document records the ownership decision for the mixed FH8626V100 Firmware preservation snapshot and the clean Firmware integration branch derived from it. It is project coordination state; do not copy it into Firmware, Builder, Linux or Divinus.

## Repositories and refs

- Firmware preservation evidence: `ArthurKoba/openipc-firmware/fh8626v100-platform@f4bf49da6ef355c9e733e00d774efe403513b1d4`.
- Firmware clean candidate: `ArthurKoba/openipc-firmware/rework/fh8626v100-clean-integration@f9146dd42a2f606d305ebccd301268848de26880`.
- Firmware base: `master@47ccdbee45fa5b8eee69c25c7af656cd5d35a28e`.
- Linux source candidate: `ArthurKoba/openipc-linux/rework/fh8626v100-final-series@357c2d13e7589db0dbe2bbf89c2ec38b1c036e6e`.
- Builder preservation/device reference: `ArthurKoba/openipc-builder/fh8626v100-anjia-ajl33pq0866@bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`.
- Divinus source candidate: `ArthurKoba/openipc-divinus/fh8626v100-canonical@1e624bd5aca97ba772413d2b00a10314d1db039f`.

## Clean Firmware decision

The clean Firmware branch is rebuilt from current Firmware `master`, not by deleting files from the WIP snapshot. Its diff contains only:

- `.github/scripts/ci-matrix.py`;
- `br-ext-chip-fullhan/board/fh8626v100/fh8626v100.generic.config`;
- `br-ext-chip-fullhan/configs/fh8626v100_lite_defconfig`.

The defconfig consumes the exact curated Linux SHA directly. No FH8626 kernel patch directory is present. The pin currently uses the ArthurKoba Linux fork because the curated series has not yet landed in `OpenIPC/linux`; this is an engineering dependency and must be replaced by the OpenIPC-owned Linux ref before an upstream-ready Firmware contribution.

The clean branch deliberately contains no AJL33PQ0866 board package or kernel fragment, no factory `.ko/.so/.bin`, no local Divinus source path and no FH8626 Divinus patch. Device policy remains a Builder responsibility; Divinus implementation remains a Divinus responsibility.

The generic Firmware config does not enable a retail-camera SD wiring option. The ANJIA Builder kernel fragment must select:

`CONFIG_FH8626V100_SD0_1BIT=y`

when it is rebuilt on top of the curated Linux source. The historical symbol `CONFIG_FH8626V100_AJL33PQ0866_MMC` is preservation-only and must not return to the clean Firmware contribution.

Firmware inherits the existing standard 8 MiB image budget: 2048 KiB kernel plus 5120 KiB SquashFS. The curated Linux source carries the matching MTD layout:

`256K boot + 64K env + 2048K kernel + 5120K rootfs + remaining NOR rootfs_data`.

On an 8 MiB NOR this leaves 704 KiB for `rootfs_data`.

The branch is a source/layout candidate, not a hardware-accepted firmware. An owner-side build must still record final `uImage` and SquashFS sizes, followed by the applicable target boot/media checks. CI registration currently marks the FH8626 family unbuilt because the kernel source is still a temporary fork pin and the defconfig builds its own GCC/musl toolchain.

## Binary inventory

All hashes below were calculated from the exact bytes stored in the Firmware preservation branch. No public Fullhan SDK/build chain has yet been established for these files, so their current provenance class is **factory/preservation evidence**, not redistributable upstream package input.

| Preservation path | Bytes | SHA-256 | Current role | Intended owner / disposition |
| --- | ---: | --- | --- | --- |
| `files/kmod/bgm.ko` | 57,812 | `715b099afb54145e11073238ecc034b3cf1c45a6e6e4253c59d887ed600bb17a` | BGM kernel media module | Linux/source replacement; otherwise evidence until defensible SDK provenance exists |
| `files/kmod/enc.ko` | 151,652 | `bdf5ce5f71bdec3dc9c7aeab4d78cfe1bb85de36f32b1104d196aaac8e911207` | encoder kernel module | Linux/source replacement; otherwise evidence |
| `files/kmod/gpio_wave.ko` | 24,704 | `58fdd10dc91a476a910758868d039b269c71986631e6c35ec9185baf45930afb` | GPIO waveform kernel module | Linux/source replacement; otherwise evidence |
| `files/kmod/isp.ko` | 144,932 | `2c77f9c294f007ef0f90d0e1855191cb24a5dea223e03d8b97c1fa5e4fd738ac` | ISP kernel module | Linux/source replacement; otherwise evidence |
| `files/kmod/jpeg.ko` | 73,856 | `1c4d45eb81b54dc19158013b3e9b9136b29f73e5140fb21fb7460a6f7c6853d2` | JPEG kernel module | Linux/source replacement; otherwise evidence |
| `files/kmod/media_process.ko` | 93,192 | `46c38614814d1dcfc507d95abaaeb01ce5235c72fab4fe8a2a2e21d75d31811d` | media pipeline kernel module | Linux/source replacement; otherwise evidence |
| `files/kmod/vmm.ko` | 15,480 | `06e07a91cf2dd8e3043769df64138160c656bd7114c76c90dc12f18fbfedd09e` | vendor memory manager | Linux/source replacement; otherwise evidence |
| `files/kmod/xbus_rpc.ko` | 23,504 | `8ffe386b7c73ca83e12582f7c306cb0e3633f9bbca39142041fc39cee2639267` | kernel RPC transport | Linux/source replacement; otherwise evidence |
| `files/lib/libmipi.so` | 5,432 | `f9260105958dd428588b0295f1db33e4fceab13e173649a8c327cd9747895b45` | userspace MIPI plug-in | Divinus/native source boundary or documented shared SDK source; not a factory blob in Firmware |
| `files/sensor/libgc1054_mipi.so` | 21,896 | `093c48710c9034dbde82681db014fd000a22f20487e74b6dbd147ebe4fff1eb9` | GC1054 userspace plug-in | Divinus GC1054 source replacement; evidence until replaced |
| `files/firmware/rtthread_arc.bin` | 232,076 | `3cfd2a04b158e624fb4c95123d0f282d261351e27337889c3d9418b552528361` | ARC/RTX firmware | Isolated firmware dependency; Firmware only if official redistributable SDK provenance is established, otherwise external evidence |
| `files/sensor/gc1054_day.bin` | 2,648 | `6d16b13a193ad44e4235550acd8b44dc7e1502bfd953b41d4e2c630495b3c337` | GC1054 day profile | Reviewed source/generated sensor data, preferably beside Divinus GC1054 implementation |
| `files/sensor/sensor_gc1054_mipi.bin` | 8,648 | `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d` | stock GC1054 sensor object | Divinus/source replacement; retained externally as stock evidence |
| `src/sensor/gc1054/profiles/gc1054_day.bin` | 2,648 | `6d16b13a193ad44e4235550acd8b44dc7e1502bfd953b41d4e2c630495b3c337` | duplicate day profile | Same payload as packaged day profile; one evidence object only |
| `src/sensor/gc1054/profiles/gc1054_night.bin` | 2,648 | `91f463a431adfd8651e5ffd3c02403f40662639d1a9cf1340358f13e921ef8f6` | GC1054 night profile | Reviewed source/generated sensor data, preferably Divinus-owned |
| `src/sensor/gc1054/profiles/gc1054_wlight.bin` | 2,648 | `b7419cef2d3c2cfe01da1eb3432a26d124ea5d1994719123959812653572a860` | GC1054 white-light profile | Reviewed source/generated sensor data, preferably Divinus-owned |

There are 16 preserved paths but 15 unique payloads because the two day-profile paths are byte-identical.

The external evidence manifest already retains `sensor_gc1054_mipi.bin` by SHA-256. The other proprietary payloads are still recoverable from the preserved Git branch but do not yet have individual external-evidence locators. Do not retire the preservation branch as an evidence source until any still-needed unique payload has been externalized with SHA-256, role/provenance and locator.

## Ownership summary

- **OpenIPC/linux:** kernel source, SoC platform code and source replacements for kernel-side FH8626 media modules.
- **OpenIPC/firmware:** generic FH8626 Buildroot/kernel-config integration and only shared runtime packages that have acceptable source/provenance.
- **OpenIPC/builder:** ANJIA AJL33PQ0866 SD/MMC selection, GPIO/PTZ/illumination/device overlay and other one-camera policy.
- **OpenIPC/divinus:** FH8626 HAL/media/ISP/sensor implementation and Divinus-specific behavior.
- **Reverse repository / evidence store:** factory binaries, stock captures, hashes, contracts and migration evidence.

No classification above is permission to delete the only known working artifact. Blob retirement still follows `docs/process/fh8626-blob-retirement.md`: replace the actual ABI/hardware contract, validate it, then remove the runtime dependency while retaining reference evidence.
