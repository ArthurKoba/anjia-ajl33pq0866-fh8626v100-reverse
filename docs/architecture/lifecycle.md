# Media-owner lifecycle and teardown contract

This document promotes the durable lifecycle boundary recovered from stock Apollo. It is independent of Majestic vs Divinus and describes ownership/recovery behavior that any replacement owner must respect.

## Accepted stock lifecycle

`HARDWARE_PASS`: the controlled stock graceful path is:

`wdt_stop -> ap_exit -> process gone -> ISP input inactive -> restart Apollo -> full pipeline returns`

This was repeated successfully across same-boot reinitialization cycles. Ordinary SIGTERM is not equivalent to this graceful path.

`video_pause` is also not global quiescence: stock ISP/statistics activity can continue while public video receive is paused.

## Static teardown order

Exact stock static reverse around `service_config_deinit @ 0xEBAFC` establishes the following dependency order:

1. guard duplicate deinit;
2. stop/wake/join service workers and optional contexts;
3. ISP/service exit stage;
4. enumerate video channels;
5. for each active channel stop VENC receive and close VPSS channel;
6. disable AI;
7. disable AO;
8. deinitialize audio core;
9. finish remaining service cleanup including the system-core exit position;
10. clear the service-running guard.

Errors are logged and later cleanup continues. A replacement should therefore use best-effort unwind rather than aborting teardown on the first destructor failure.

## Replacement owner states

Recommended explicit lifecycle states:

`UNINITIALIZED -> OWNER_ACQUIRING -> OWNER_ACQUIRED -> ISP_READY -> CHANNELS_READY -> STREAMING`

Runtime/teardown substates should distinguish at least:

- `RECEIVE_PAUSED`;
- `ALGO_PAUSED`;
- `QUIESCING`;
- `QUIESCED`;
- `STOPPING`;
- `STOPPED`;
- `POISONED` for an unrecoverable partial teardown.

Do not collapse pause, quiesce and teardown into one boolean.

## Production shutdown requirements

A safe replacement shutdown should:

1. explicitly safe board outputs (white light off, PTZ stop, speaker mute policy);
2. reject new control transactions;
3. freeze algorithm commits and drain in-flight work;
4. release outstanding stream descriptors;
5. stop VENC receive and close VPSS channels;
6. stop audio workers/AI/AO;
7. stop pack/mux workers;
8. tear down ISP/media/system ownership;
9. unmap/free/close replacement-owned resources;
10. verify no active descriptors/commits/ISP input remain;
11. enter `STOPPED`; a new start creates a new owner generation.

Every acquired resource should enter a ledger immediately so partial initialization can unwind in reverse dependency order.

## Reloadable callback / stream-safety boundary

A historical native-owner audit exposed three implementation hazards that are not stock requirements but are durable replacement-design constraints:

- unloading/replacing an algorithm module without a quiesce/generation barrier can leave `algo_tick`, sensor callbacks or re-entrant users executing code/data after `dlclose`;
- validating a 32-bit stream/ring address with `ring_base + ring_size` in 32-bit arithmetic can wrap near the address-space limit;
- using a 32-bit capture-byte counter against a 64-bit capture limit can silently wrap when limits are raised.

Production replacement code should therefore:

- quiesce all callback users before `fini`/`dlclose`, advance a callback generation, then publish the new callback set;
- widen ring-end and tail arithmetic to at least 64 bits before bounds comparison/copy calculations;
- keep capture counters and limits in the same sufficiently wide integer domain.

These are source-level safety lessons from an obsolete owner snapshot, not evidence that stock Apollo contained the same bugs.

## Negative contracts

Stock behavior does **not** guarantee all safety properties a replacement may want:

- graceful `ap_exit` does not guarantee white-light/PWM safe-off;
- staged IQ publication is not a universal all-consumer atomic transaction;
- software/controller state may outlive recreated hardware state;
- `video_pause` does not prove a quiesced ISP/media pipeline.

OpenIPC may deliberately provide stronger behavior, but should label it as replacement policy rather than stock equivalence.

## Evidence boundary

Lifecycle/reinit is closed for implementation. Reopen only for a concrete contradiction in target behavior or a replacement-owner failure that cannot be explained by the known ownership/teardown contract.
