# Stock restore safety

Historical source date: 2026-09-15.

## Mandatory rule

Stock restore commands must stop after the final `sf write`.

`reset`, boot, or power-cycle is always a separate explicit operator action. Do not combine restore and reset inside `setenv exec ...; run exec` or any equivalent one-line macro.

Recovery commands must remain safe to interrupt between flash-write completion and any reset/boot action.

## Why this exists

A historical recovery command chained flash write and `reset` together. The workflow was interrupted and the automatic reset occurred at an unsafe point. The rule above supersedes that pattern.
