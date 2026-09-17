# FH8626V100 U-Boot port findings

This document records the strongest camera-level U-Boot acceptance retained for AJL33PQ0866. Implementation source authority is the relevant `ArthurKoba/u-boot-fullhan` branch; the known engineering ref is recorded in `docs/process/upstream-integration.md`.

## RAM-chainload baseline

`HARDWARE_PASS`: a current-upstream U-Boot port was chainloaded non-persistently from stock U-Boot and exercised on ANJIA AJL33PQ0866 before bootloader flash integration.

The validated RAM path proved:

- entry into U-Boot 2026.10-rc3-class current upstream on ARM1176;
- UART console;
- 64 MiB DRAM discovery and relocation;
- timer/autoboot behavior;
- both FH8626 GPIO banks;
- GMAC MDIO, PHY identification, link negotiation and working network/TFTP data path;
- SPI NOR identification/read path;
- legacy image verification and Linux kernel handoff.

The RAM target intentionally depended on vendor-initialized DDR/pinmux/clock state. Passing the chainload target therefore proved the reconstructed runtime peripheral contracts, not independent cold-boot/bootstrap ownership.

## SPI NOR wrapper and DMA contract

Focused stock reverse corrected the Fullhan DesignWare SSI wrapper configuration for the NOR path. The proven wrapper programming preserved bit 7 and selected the stock NOR mode corresponding to:

`CCFGR = (CCFGR & 0xFFFFF880) | 0x200E`

The correctness fallback bounded CPU-polled reads to the detected RX FIFO depth. A later stock-derived RX-DMA path used the FH8626 DW AHB DMAC and retained automatic fallback to bounded FIFO reads on timeout.

`HARDWARE_PASS`: at 50 MHz SPI clock, a complete 3 MiB kernel-partition read reproduced target CRC32 `669b23f3`, passed legacy-image checksum verification, and repeated successfully after zeroing the destination range. The run showed no DMA-timeout/fallback message, so the eligible transfer exercised the DMA fast path.

## Vendor-compatible `kload`

`HARDWARE_PASS`: the compatibility command performed the stock-layout read:

- SPI NOR bus 0 / chip-select 0;
- flash offset `0x00050000`;
- length `0x00300000`;
- destination `0xA1000000`.

The standalone command reproduced CRC32 `669b23f3`, identified the expected `Linux-4.9.129-fh8626v100` legacy image and passed checksum verification. It was read-only: no environment persistence or flash write was part of this gate.

## Safety and persistence boundary

The bring-up deliberately separated RAM-chainload proof from bootloader flash deployment. Do not reinterpret the hardware-pass RAM sequence as proof that every later cold-boot/bootstrap integration state was accepted.

The known engineering branch advanced beyond the original RAM-only checkpoint. Before future implementation work, verify its current remote state and curate accepted changes from the appropriate current base rather than treating preservation history as upstream-ready.

## Reverse boundary

The exact stock U-Boot partition is a 196,608-byte ARMv6 little-endian reverse input loaded historically at `0xA0800000`; analysis recovered hundreds of functions and the contracts summarized above.

Any renewed low-level U-Boot reverse is performed in the canonical Ghidra MCP project. Local Ghidra project paths, export trees and historical filesystem layouts are not part of the current camera repository contract.
