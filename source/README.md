# Camera-specific source

`source/` contains camera-specific components, contracts and source-level tests that are useful inputs for integration into the OpenIPC repositories that own production code.

The curated FH8626V100 material is under `fh8626v100/components/`.

The directory is intentionally component-oriented rather than a monolithic firmware/application tree. Each retained unit should be independently useful for reuse, validation or curation into its owning repository.

Run the retained host/source checks with:

`make -C source/fh8626v100/components check`

Keep source-level tests and contracts when they materially describe or validate behavior. Working reverse reconstructions belong in Ghidra MCP; build outputs, caches, probes and generated analysis do not belong here.

Source or host-test coverage does not imply target hardware acceptance.
