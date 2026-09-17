# Stock `dev_ctrl` role and command plane

`dev_ctrl` is a stock board/service-control daemon. Exact unpacked target reverse closes its role as separate from the media owner and watchdog feeder.

## Role classification

`REVERSE_CONFIRMED`: `/app/abin/dev_ctrl` owns local board-service commands for SD/MMC, network/Wi-Fi, GPIO/board utilities, MTD/U-Boot environment handling, administrative shell/reboot and firmware-update safety.

It is **not** the hidden Apollo/media owner, ISP owner, watchdog feeder or Apollo restart supervisor.

This negative ownership result is independently consistent with runtime evidence: Apollo owns `/dev/watchdog`; `dev_ctrl` is a separate PID1 child and survives controlled Apollo watchdog stop/start and `ap_exit` cycles.

## IPC planes

Exact static reverse recovers two `dev_ctrl`-owned local UNIX-domain command servers:

- `@jdc_server` — device-command / `apcmd` JSON plane;
- `@jsh_server` — shell-command / `shcmd` JSON plane.

The generic JCMD library also contains `@jap_server` as a default application-server name, but this exact `dev_ctrl` main does not create it.

The generic server object uses a callback, synchronization/thread object, running flag, local socket fd, receive buffer pointer and 1024-byte receive size.

## Command-table ABI

Each command record is 20 bytes:

- command-name pointer;
- minimum argc;
- maximum argc (`-1` means unbounded upper count in this build);
- handler pointer;
- help-string pointer.

The dispatcher supports help/list/search, argument-count validation and a shell-execution escape path.

Recovered public commands:

| command | handler | role |
|---|---:|---|
| `sd_check_plug` | `0x11FCC` | SD plug-state query |
| `sd_hotplug` | `0x11F88` | software SD hotplug |
| `sd_plug` | `0x11F0C` | software plug in/out |
| `sd_power` | `0x11E90` | SD power control |
| `netcmd` | `0x12044` | network interface/Wi-Fi control |
| `sd_is_formatting` | `0x128AC` | SD formatting-state query |
| `sd_format` | `0x1294C` | SD format command |

The lower service layer also contains SD event notification, network/Wi-Fi helpers, MTD/environment utilities and guarded reboot/update operations.

## Reboot boundary

`dev_ctrl` imports `reboot()` and contains administrative reboot infrastructure. That does not make it the watchdog/reset owner. Its explicit reboot path is board-service policy and is separate from:

- Apollo watchdog expiry;
- kernel/PMU restart;
- graceful `ap_exit`;
- media-owner lifecycle.

The stock guard `In FW upgrading, can't reboot` demonstrates that administrative reboot is suppressed while firmware upgrade state is active.

## OpenIPC consequence

No `dev_ctrl` binary compatibility layer is required for native FH8626 media/HAL ownership.

Required board services should be provided by normal OpenIPC/platform components:

- storage/hotplug service;
- network service;
- GPIO/platform layer;
- MTD/environment tooling where intentionally exposed;
- controlled system reboot/update manager.

Keep these board services separate from the single media hardware owner and watchdog supervisor.

## Closure

`STATIC_ROLE_CLOSED / BOARD_SERVICE_DAEMON / NOT_MEDIA_OR_WATCHDOG_OWNER`.

Further reverse of miscellaneous helpers is historical archaeology unless a concrete board-service implementation blocker appears.
