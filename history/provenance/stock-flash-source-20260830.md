# Stock flash provenance

The authoritative stock-flash acquisition is the retained 8 MiB MX25L6405D full-flash dump indexed in `evidence/MANIFEST.tsv`:

- SHA-256 `3f0c59ff54cba59c62a6bb971409f075a73e456c11684ea925b9df3b891fe1c8`;
- Drive locator `1A8Eeg5JcO7epvk6H8P-9w9-739xVomAt`.

Partition-sized bootstrap/U-Boot/kernel/data/res/app slices are derivable views of that acquisition and are not independent primary stock authority.

A later standalone bootstrap/MTD capture differed from the corresponding region of the pre-reverse full flash. Because later per-MTD captures could reflect modified development state, they must not override the full-flash acquisition when reconstructing stock state.

Runtime RAM/MMIO/VMM/RAW/UART captures remain independent primary evidence because they are not reconstructable from flash.
