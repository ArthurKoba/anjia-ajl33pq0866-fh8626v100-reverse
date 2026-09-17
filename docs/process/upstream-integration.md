# Related repository integration

This document records known FH8626V100 engineering refs that can be used as starting locators when the related OpenIPC repositories are reconciled.

The refs are not assumed to be the current integration bases. Verify each remote repository when that phase begins.

## Known refs

- `ArthurKoba/openipc-builder`
  - branch `fh8626v100-anjia-ajl33pq0866`
  - known engineering tip `bcf8658e4aa612ee9afda8c28d52d8ad1674e2f1`
  - earlier checkpoint `5603a701c8812aebc705c42e933ebae48aed805f`
- `ArthurKoba/openipc-divinus`
  - branch `fh8626v100-canonical`
  - known engineering tip `1e624bd5aca97ba772413d2b00a10314d1db039f`
- `ArthurKoba/openipc-firmware`
  - branch `fh8626v100-platform`
  - known engineering tip `6db66c53971fda8ba733f370a965e52cd53fb61b`
- `ArthurKoba/openipc-linux`
  - branch `fullhan-fh8626v100`
  - known engineering tip `ebf5d776c748edbd58c1aaf8be9d5b2639a16834`
- `ArthurKoba/u-boot-fullhan`
  - branch `fh8626v100-mainline`
  - known engineering tip `ae63365e10b38e5b9ed3bd5173a0ed3e5c8f9996`

## Integration rule

For each component repository:

1. inspect the current repository/upstream base;
2. compare existing FH8626 work against the camera contracts in this repository;
3. retain only changes that remain technically relevant;
4. remove generated binaries, debug scaffolding and obsolete experiments from the contribution set;
5. rebuild accepted changes on a clean working branch from the verified target base;
6. group work into coherent, reviewable commits;
7. build/test the curated result at the appropriate evidence level;
8. leave final PR creation to the repository owner.

## Authority

This camera repository owns camera-level hardware/media facts. Component repositories own their implementation code. If integration work reveals a new camera contract, update the relevant camera documentation here and then implement against that contract in the component repository.
