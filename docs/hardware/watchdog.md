# Watchdog and recovery contract

This file records the accepted FH8626V100 watchdog behavior and OpenIPC implementation boundary for AJL33PQ0866.

## Hardware and driver boundary

The target uses a Synopsys DesignWare watchdog with FH8626 PMU integration. Stock/runtime reverse established application-managed ownership through `/dev/watchdog`; conventional desktop-Linux assumptions about sysfs/timeleft interfaces are not sufficient evidence by themselves.

A requested 30-second stock timeout was observed to map to approximately 33.55 seconds. Implementations should read back the actual programmed timeout and schedule feed/health logic from that value rather than assuming the request was exact.

`WDIOC_SETOPTIONS(WDIOS_DISABLECARD)` plus PMU pause is the meaningful deliberate-disable path. Increasing the timeout, writing the magic-close character and closing the descriptor does not safely disable this target's hardware watchdog.

## Accepted OpenIPC feeder baseline

`HARDWARE_PASS`: the exercised board integration used an early exclusive `/dev/watchdog` owner through `/etc/init.d/S01watchdog`, running BusyBox as:

`/sbin/watchdog -F -T 15 -t 5 /dev/watchdog`

The init script backgrounded that foreground process, recorded `/run/watchdog.pid`, and intentionally left the feeder alive during the rcK stop pass so an orderly reboot did not close the watchdog prematurely.

Acceptance showed:

- the feeder survived beyond the configured timeout while feeding;
- its open descriptor resolved to `/dev/watchdog`;
- killing the feeder reset the camera;
- the following boot automatically started a new feeder.

This establishes protection against a stalled/dead userspace feeder and whole-kernel stalls for the exercised baseline.

## Preferred product architecture

A future richer supervisor should remain the single long-lived watchdog owner and separate:

- hardware feed cadence;
- named service-health evaluation;
- recovery grace duration;
- persistent reset-reason logging.

Useful reset-reason categories include operator reboot, software fatal, health-gate timeout, watchdog-feed I/O failure, kernel panic, and unknown/power loss. Stock does not provide a useful hardware boot-status reason, so intentional reset reasons must be persisted by OpenIPC before reset.

For deliberate recovery, first make illumination, audio output, PTZ and storage safe, then use orderly reboot or intentionally cease feeding at the normal timeout. Do not use max-timeout/magic-close as a prompt recovery mechanism.

## Acceptance boundary

The simple BusyBox feeder is hardware-accepted and sufficient to keep the watchdog armed. A health-gated supervisor and persistent reset-reason system are future product improvements, not prerequisites for the existing hardware pass.
