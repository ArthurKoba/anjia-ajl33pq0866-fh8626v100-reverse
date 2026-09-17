# Engineering dead ends and durable lessons

This file keeps only camera-level mistakes and rejected directions that are still useful. Repository migration, patch-staging and temporary toolchain/process mistakes are intentionally excluded.

## Homologs are not ABI donors

Family similarity across Fullhan SoCs is useful for architecture and naming, but does not prove FH8626 structure layouts, register offsets or ioctl contracts.

Correct rule: FH8626 ABI/register claims require FH8626 target evidence. Homologs are semantic/structural references only.

## Host/build success is not hardware acceptance

A successful host test, ARM build, stub path or synthetic RTSP test proves only the layer it exercised.

Correct rule: keep source/host/target/hardware evidence classes separate and do not promote one into another.

## Vendor wrappers may own hidden state

Direct GC1054 callback invocation was once treated as equivalent to `API_ISP_SensorInit()`.

Reverse evidence showed the stock wrapper also acquires/masks chip ID and propagates state before invoking sensor initialization.

Correct rule: do not replace a vendor wrapper with its apparent terminal callback until all side effects and ordering dependencies are proven equivalent.

## Exact recovered arithmetic can depend on pipeline context

An isolated replay of historical D0238 output words did not remove a visual artifact even though later reconstruction reproduced the stock words exactly.

Correct rule: failure of a detached register replay does not invalidate an exact recovered subroutine when the original runtime ordering/context is absent.

## Live U-Boot environment outranks compiled defaults

Compiled U-Boot strings for `mtdparts`, `bootcmd` and related values differed from the captured effective environment.

Correct rule: use captured bootstrap/environment state and observed UART behavior as authority for the live camera configuration.

## Do not assume standard watchdog interfaces

The stock target exposed `/dev/watchdog` with Apollo ownership but no conventional `/sys/class/watchdog` control tree.

Correct rule: identify the real target interface and owner before designing control or validation around desktop-Linux conventions.

## Persuasive labels do not make valid reverse evidence

Some targeted address slices were saved under plausible watchdog/restart names but decoded data as code and showed incoherent control flow.

Correct rule: require coherent function boundaries plus xref/string/symbol/data-flow support before promoting an address range to a recovered function. Use the canonical Ghidra MCP project for such analysis.

## Positive event capture does not prove exclusivity

The accepted PTZ ioctl capture proved that observed LEFT/UP events used `/dev/fh_pwm` and request `0xC004700C` with specific payloads.

Correct rule: narrowly filtered positive evidence establishes what occurred in the captured event; it does not prove that no alternate path exists.

## Low-yield sampling is inferior to causal event evidence

Long generic syscall snapshots around short PTZ events produced large amounts of repeated idle state and little causal information.

Correct rule: for short hardware-control events, prefer focused event evidence and corroborate with subsystem counters or direct target state.

## Conventional distro layout assumptions are unsafe

The stock embedded rootfs did not expose several paths/tools common on general Linux systems.

Correct rule: inventory the actual target filesystem, device nodes and kernel/runtime surfaces before concluding that a capability is absent.

## Target wall clock is not project chronology

One stock capture reported a stale wall-clock date inconsistent with the real project session.

Correct rule: use operator/session provenance for historical dating; target wall clock is runtime state unless independently synchronized.

## General rule

Exact target hardware/runtime evidence outranks historical prose. Static reverse, source implementation and runtime observation each keep their own evidence boundary.
