# Sensitive-Data Handling

This repository is **public**. The following classes of data were scrubbed or excluded before publication. If you fork/extend this, keep them out.

## Excluded entirely (never committed)

| Item | Why |
|---|---|
| `ODIN3_FOLDERS/` — AYN factory firmware (signed `xbl*`, `abl`, `tz`, `featenabler.mbn`, `super`, `boot`, `vbmeta`, `NON-HLOS`, `dspso`, …) | Proprietary AYN/Qualcomm copyright. Documented and **hashed** in the docs; binaries never published. |
| Third-party **donor firmware** (Realme GT7 Pro, Xiaomi 15 Ultra, iQOO 13, OnePlus, RedMagic) | Proprietary; used only for offline comparison. Referenced by source, not redistributed. |
| Raw partition dumps — `super_*.img`, `vendor_dlkm*.img`, `vbmeta*.img`, `boot*.img`, EDL backups | Contain proprietary firmware and/or device-unique data. |
| Firehose programmers / loaders (`*firehose*`, `prog_firehose*`, `*.melf`) | Signed Qualcomm/OEM tooling. |
| Any `*.ko` / `*.so` / `*.dtb` / `*.apk` / `*.dex` binaries | Build outputs / proprietary blobs. |

These are blocked by [`.gitignore`](.gitignore) by default.

## Redacted in text/evidence

| Item | Treatment |
|---|---|
| Device serial (the `adb`/`fastboot` serial) | Replaced with `<SERIAL>` in published evidence. |
| IMEI / MEID / SIM / `serial_number` socinfo field | Replaced with `<REDACTED>`. |
| Personal filesystem paths (local user home dir) | Generalized to `<repo>/…` or `<artifacts>/…`. |
| Any keys, certs, tokens, signatures | Excluded; only public register/fuse *names* are kept. |

## What IS published (safe)

- All **original analysis, documentation, methodology, and conclusions**.
- **Scripts** authored for the investigation (read-only probes, decode helpers) — with device-unique values templated out.
- **Curated text evidence**: on-device read outputs, `avbtool` info dumps, e2fsck/dumpe2fs output, decode reports, and **SHA-256 hashes** of the firmware (hashes are safe to publish; the files are not).
- Quoted firmware **strings and register/fuse names** (e.g. `NSP_DISABLE`, `qsee_is_sw_fuse_blown`, `0x221c2420`) under fair-use for interoperability/research documentation.

## If you reproduce this

Use the read-only steps in [`docs/07-reproduce`](docs/07-reproduce/). Do **not** publish your device serial, IMEI, or any firmware you pull. The findings are reproducible without redistributing any proprietary binary.
