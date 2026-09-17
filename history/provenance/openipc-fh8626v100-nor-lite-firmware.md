# FH8626V100 OpenIPC NOR-lite firmware provenance

This record identifies historical FH8626V100 OpenIPC firmware pairs retained as binary evidence. It does not make either pair current product authority or hardware acceptance.

## Retained pair

- `rootfs.squashfs.fh8626v100`
  - SHA-256 `944109493eb25c006d6b334214da3451946ef95655838e6e3bd08a94ec0e0c81`
  - size 2,863,104 bytes
- `uImage.fh8626v100`
  - SHA-256 `9058d801bf9a9c43fb89cff7009fc8ea2a83797e647ae7b936614df720ad10b5`
  - size 1,588,224 bytes
  - image name `Linux-4.9.129-fh8626v100`

## 2026-09-03 MAC-address integration variant

A second byte-distinct pair was retained from the MAC-address integration phase:

- rootfs SHA-256 `5aa507a6cec537610bab99228492d495f0cb312351805b468f1a9a0df20c5fa1`, 1,675,264 bytes;
- uImage SHA-256 `fdddb81a2cbe43da290c9d2da15766d12113a1ff694fb38b1841d02dd958fd5e`, 1,581,344 bytes, load/entry `0xA0008000`.

These hashes are provenance identities only. Current firmware work belongs in the owning OpenIPC repositories and must be rebuilt/revalidated from the selected current source refs.
