# FH8626 Divinus correction ledger

Purpose: record concrete FH8626 contract mistakes or unsafe assumptions found while independently rebuilding the Majestic compatibility path. This is a correction queue for the Divinus agent, not a Majestic design document.

Evidence priority:
1. Ghidra-recovered stock Apollo / kernel module behavior;
2. hardware-proven runtime evidence;
3. current source implementations only as secondary references.

## Confirmed corrections

### VPSS Enable payload semantics

Status: CONFIRMED.

Stock Apollo wrapper `FH_VPSS_Enable(channel)` passes the **channel id itself** as the 4-byte payload to ioctl `0xC004694D`.

The payload is not a boolean enable flag.

Evidence:
- Apollo wrapper used by `service_vi_start()` passes channel 0 directly.
- `isp.ko:vpu_control` dispatch for `0xC004694D` copies one 32-bit word and calls `vpu_enable(channel)`.

Correction:
- any Divinus implementation that sends `1` for channel 0 or treats the request as `enable=true` is wrong.
- represent the API as `enable(channel)`.

### VPSS Disable is a distinct ioctl

Status: CONFIRMED.

`isp.ko:vpu_control` dispatches ioctl `0xC004694E` directly to `vpu_disable()`. It takes no userspace payload.

Correction:
- do not model disable as `0xC004694D` with payload 0.
- use the distinct native request.

### VPSS CloseChn

Status: CONFIRMED.

Stock Apollo `FH_VPSS_CloseChn(channel)` issues `0xC0046950` with a 32-bit channel payload. `isp.ko` copies that word and calls `vpu_close_chn(channel)`.

Correction:
- replace any teardown stub or guessed enable/disable alias with the real CloseChn operation.

### VPSS frame-control wire

Status: CONFIRMED.

Stock Apollo `FH_VPSS_SetFramectrl(channel, pair)` accepts two 16-bit values, requires both nonzero, packs them into one 32-bit word, and sends:
- word0 = channel
- word1 = low16 numerator / high16 denominator
- ioctl `0xC0086954`

`isp.ko:vpu_set_frm_ctrl_cfg` confirms this exact two-word wire and validates channel/configured state plus ratio limits.

Correction:
- do not leave frame control as a stub.
- do not reinterpret it as an arbitrary packed-fps word without preserving the public `uint16_t[2]` contract.

### VENC configuration wire

Status: CONFIRMED KERNEL SIDE.

`enc.ko:pae_enc_set_config` receives exactly 11 words through `0xC02C5006`.

Known fields:
- word0 channel
- word1 visible width
- word2 visible height
- word4 H.264 profile (`0x42` Baseline, `0x4d` Main)
- word5 init QP, valid 0..51
- word6 packed frame ratio; both 16-bit halves must be nonzero

The driver aligns capacity internally and generates crop/SPS/PPS itself.

Correction:
- do not pass aligned 1088 as visible 1080 height.
- do not synthesize SPS crop externally where the driver already owns it.
- keep visible geometry separate from allocated/aligned capacity.

### VENC RC wire

Status: CONFIRMED KERNEL SIDE.

`enc.ko:pae_ioctrl`:
- full RC set = `0xC054502F`, total wire 0x54 bytes
- full RC get = `0xC0545030`, total wire 0x54 bytes
- realtime RC change = `0xC01C5055`, 0x1c bytes

`pae_set_rc_cfg` treats the first word as channel and copies the following 0x50 bytes into channel state after rate-control initialization.

Correction:
- keep full and realtime RC records separate.
- do not collapse them into one guessed runtime-control struct.

### VENC lifecycle order

Status: CONFIRMED STOCK APOLLO ORDER.

For a normal H.264 channel, Apollo service startup performs:

`VPSS SetChnAttr -> VPSS OpenChn -> VPSS SetFramectrl -> VENC CreateChn -> VENC SetChnAttr -> VENC StartRecvPic -> SYS BindVpu2Enc`

VI startup performs `FH_VPSS_Enable(channel)` separately before the VENC service sequence.

Correction:
- Divinus lifecycle should not assume bind necessarily precedes StartRecvPic.
- VI/VPSS global/channel enable must remain separate from VENC start ownership.
- teardown should mirror the actual ownership/state transitions rather than only reversing an inferred simplified sequence.

### VENC CreateChn public record

Status: CONFIRMED STOCK APOLLO PUBLIC API.

Apollo `FH_VENC_CreateChn(channel, attr)` consumes exactly the first three words as:
- support_type bitmask
- capacity width
- capacity height

For H.264:
- bit 0x4 = normal H.264
- bit 0x8 = smart H.264

The service computes memory requirements for every selected encoder capability before allocating channel memory.

Correction:
- do not ignore CreateChn attributes and silently create a fixed 720p channel.
- validate the requested support type and capacity before channel setup.

### VENC stop / stream release

Status: CONFIRMED.

`enc.ko:pae_enc_stop(channel)` clears running state and waits for encoder drain.

Encoded stream release ultimately calls `media_stream_release(4)` and requires the encoder channel to be configured.

Correction:
- preserve balanced stream ownership.
- release outstanding stream state before final encoder teardown.
- do not treat StopRecvPic as an instantaneous flag-only operation.

## Audio observations requiring Divinus review

Status: CONFIRMED STOCK APOLLO HIGH-LEVEL LIFECYCLE.

Apollo `FH_AC_Init()` does more than low-level RTX initialization:
1. RTX init;
2. board DSP init blob;
3. optional AEC config;
4. capture NR policy;
5. playback NR policy;
6. initialized-state publication.

Then service startup builds a seven-word public audio config:
`type, channels, sample_width, sample_rate, packet_bytes, encoding, volume`

and calls:
`FH_AC_Set_Config -> FH_AC_AI_Enable -> FH_AC_AI_SetVol`.

Correction/review:
- compare Divinus audio start with this full stock lifecycle.
- do not assume raw RTX init alone is equivalent to public `FH_AC_Init`.
- board init blob / NR policy must have explicit ownership and evidence.

## Open review items

These are not yet correction claims; they are active comparisons:
- exact public `FH_VENC_SetChnAttr` -> 11-word PAE translation;
- exact producer/MMIO gate ownership relative to VI start, VENC start and bind;
- full ISP initialization and frame-loop sequencing;
- same-boot ISP teardown/restart;
- JPEG public API translation;
- final audio AO/mute/two-way ownership.

Update this ledger only when a mismatch is evidenced by stock Ghidra, runtime evidence, or an authoritative platform contract.
