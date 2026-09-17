# Operator interaction protocol

The human operator executes commands on the camera/WSL/Windows host and handles physical hardware. The agent owns technical analysis, experiment selection, interpretation, state transitions and documentation.

## Command UX

- Send one coherent command block when no intermediate output is needed.
- When the next action depends on command output, stop at that diagnostic checkpoint and inspect the returned result before continuing.
- State the execution environment when it matters: `CAMERA`, `WSL` or `WINDOWS`.
- Use an explicit `cd` or absolute path when commands depend on the current directory.
- Prefer a transferred archive/finished file over lengthy manual file creation when several files are involved.
- Avoid defensive preflight checks that do not affect the next decision; try the reasonable action and diagnose the actual failure if it occurs.
- The agent, not the operator, chooses technical branches after interpreting results.

## Hardware experiments

Keep causal state explicit. A useful pattern is:

`BOOTED -> OWNER_READY -> BASELINE_VALIDATED -> CHANGE_APPLIED -> CHANGE_VALIDATED -> RESTORED -> COMPLETE`

After reboot, rediscover volatile process/fd/runtime state before relying on it.

Before a reboot-prone experiment, preserve unique evidence that would otherwise exist only in volatile storage.

## Hardware acceptance

Build success, a GPIO latch or a log line saying `success` is not by itself physical proof. Use the observable required by the feature: image/FOV change, audio output, movement, network traffic, register/readback state or another explicit target criterion.

## Writes and rollback

For a destructive or stateful experiment, define the intended baseline, minimal write/change set, success observable and practical restore path before applying it. Verify restoration when failure to restore would affect later tests.

## Scope

This document governs interaction and experiment UX only. Subsystem evidence and current technical contracts remain authoritative for camera behavior.
