# GC1054 profile payloads

These three 2648-byte files are exact profile payload slices extracted from the retained stock GC1054 vendor container `sensor_gc1054_mipi.bin`.

Container identity:

- external evidence SHA-256: `92a0289bb7e5774af1210458588815358192aae3c29f59e15787249724c5428d`
- Drive locator: `10tFZJTUrqLAyNKA19MYEGJD570TeRQcP`
- size: 8648 bytes
- manifest role: `stock-gc1054-sensor-binary`

Exact payload mapping:

| file | container offset | size | SHA-256 |
|---|---:|---:|---|
| `gc1054_day.bin` | `0x02C0` | 2648 (`0xA58`) | `6d16b13a193ad44e4235550acd8b44dc7e1502bfd953b41d4e2c630495b3c337` |
| `gc1054_night.bin` | `0x0D18` | 2648 (`0xA58`) | `91f463a431adfd8651e5ffd3c02403f40662639d1a9cf1340358f13e921ef8f6` |
| `gc1054_wlight.bin` | `0x1770` | 2648 (`0xA58`) | `b7419cef2d3c2cfe01da1eb3432a26d124ea5d1994719123959812653572a860` |

The Git blobs were verified byte-for-byte against those container slices during repository cleanup.

These are small reusable camera-profile inputs, not a second copy of the vendor library/container and not generated build output. The complete original vendor container remains external evidence; if profile semantics change, derive them again from the retained container rather than editing these payloads ad hoc.


A consumer that already accepts a raw 0xA58 ISP parameter payload may use
`gc1054_day.bin` directly. It does not need to reconstruct or ship the full
8648-byte SREG container merely to obtain the active day profile. Container
parsing remains appropriate only for code whose API explicitly expects the
multi-profile SREG container.
