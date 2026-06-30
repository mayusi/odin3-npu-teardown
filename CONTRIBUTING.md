# Contributing

This repository is primarily a **finished investigation record**. Contributions are welcome in the spirit of correcting, extending, or independently reproducing the findings.

## Good contributions

- **Independent reproduction** of the read-only findings on another Odin 3 / CQ8725S unit (especially a *different SKU* or production firmware — see below).
- **Corrections** to any technical claim, with evidence.
- **New data** that bears on the residual ~6% uncertainty — most valuably, a genuine read of QFPROM `0x221c2420` from a `peek`-capable production-signed Firehose loader, or a TZ-side fuse read.
- **A different-SKU data point**: if you have an Odin 3 (or sibling device) whose `/sys/devices/soc0/nsp` reads `0x0`, that would be a major finding — please open an issue with the full `soc0` dump (scrub your serial/IMEI).

## Hard rules (so the repo stays publishable)

1. **Never commit proprietary firmware binaries** — no `*.img`, `*.elf`, `*.mbn`, `*.melf`, signed loaders, donor firmware, or partition dumps. The [`.gitignore`](.gitignore) blocks these by default. Document and hash them instead.
2. **Scrub device-unique data** — serial, IMEI, `serial_number`, keys, personal paths. See [`SENSITIVE.md`](SENSITIVE.md).
3. **Read-only only** in any "how to reproduce" content. This project's brick (documented in [the brick page](docs/03-methods/08-brick-and-recovery.md)) is exactly why: flashing modified verity-backed partitions bricks the device.
4. **Cite real evidence.** Claims reference files in [`evidence/`](evidence/) or quote real values.

## Repro environment

- `adb` (socinfo reads need no root). Optional root via your own method for the deeper read-only probes.
- Linux tooling (`avbtool`, `dumpe2fs`, `debugfs`) for the offline AVB/ext4 analysis, if reproducing the postmortem.
