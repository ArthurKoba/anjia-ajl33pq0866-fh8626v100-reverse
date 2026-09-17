# Network and removable-storage contracts

This document records camera-level network/storage contracts for ANJIA AJL33PQ0866. Historical implementation policy is not automatically current product policy.

## Ethernet identity and network acceptance

`HARDWARE_PASS` established a persistent locally administered Ethernet identity `02:06:c7:6a:5a:1a` on the exercised camera.

The accepted integration normalized that value into standard U-Boot `ethaddr`, passed it on the kernel command line and populated FH8626V100 GMAC platform data before device registration. Three complete boot cycles retained the same MAC, carrier, IPv4 reachability and SSH behavior without relevant GMAC/PHY/DMA errors.

A later integration run obtained a normal DHCP lease and installed wired routing/DNS successfully. Any specific lease address from that run is historical router state, not a compiled camera address. Recovery/fallback IP policy must be treated separately from normal DHCP product behavior.

The attached RTL8188FU USB Wi-Fi module exposed physical MAC `94:a4:08:66:7b:fa`. In the exercised no-credentials state, `wlan0` stayed down and neither `wpa_supplicant` nor `hostapd` started. Wired network availability was independent of Wi-Fi presence.

Current implementation source belongs to the related Builder/Firmware/Linux repositories; known engineering refs are listed in `docs/process/upstream-integration.md`.

## AJL33PQ0866 microSD hardware contract

`HARDWARE_PASS` established the board's SD0 path:

- controller resource `0xe2000000..0xe2003fff`;
- stock raw IRQ14, observed through Linux IRQ30 on the accepted kernel;
- one-bit SD0 wiring using card-detect, clock, command/response and DATA0;
- pad66 remains available as active-low GPIO61 reset input because DATA1 is not used by the selected one-bit profile;
- product reset observation was release/press/release `1/0/1` on GPIO61;
- controller card-detect/write-protect state is used rather than an invented board CD/WP/power GPIO callback.

The accepted target run enumerated `mmc0`, a 972 MiB high-speed card and `mmcblk0p1`; CID/CSD were readable, a bounded sector-zero read succeeded, and no MMC CRC/timeout/I/O errors were reported.

## Storage safety

The exercised runtime mounted the card read-write. A dirty FAT warning was observed, so the durable safety rule is:

- do not run repairing `fsck.fat` while the filesystem is mounted;
- stop local writers and unmount or remount read-only before offline integrity/repair;
- orderly service shutdown should close media files and sync before software unmount;
- sudden card removal or power loss cannot be made transactionally safe by userspace alone.

Storage hardware and shutdown-safety contracts are streamer-independent.

## Device-profile boundary

Board-specific SD wiring, Wi-Fi selection, PTZ package and AJL33PQ0866 product configuration belong to the named camera/device profile rather than generic FH8626V100 support.

The exact implementation shape is owned by the related OpenIPC repositories. This repository records the camera contract and accepted target behavior.
