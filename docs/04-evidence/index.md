# Evidence Index

The `evidence/` folder contains the raw artifacts — on-device reads, decode reports, postmortem analyses, build logs, and SHA-256 hashes — that back every factual claim in this repository. This page explains the folder structure, what is included and excluded, and maps each evidence subfolder to the documentation that cites it.

For the full scrubbing and exclusion policy, see [SENSITIVE.md](https://github.com/mayusi/odin3-npu-teardown/blob/main/SENSITIVE.md).

---

## Folder Layout

```
evidence/
├── fuse-reads/              ← Live on-device sysfs and iomem dumps (read-only, uid=0)
├── avb-verity/              ← avbtool info, vbmeta descriptor dumps, verity digest comparison
├── vendor-dlkm-postmortem/  ← e2fsck / dumpe2fs output, dm-verity error trace, recovery log
├── xbl-boot-policy/         ← XBL disassembly excerpts (fuse-read rule, NSP gate), annotated
├── edl-firehose/            ← EDL session transcript, Firehose response log, backup manifest
├── kernel-build/            ← Build log excerpts, module list, patch diffs (log-only)
├── feature-enabler/         ← featenabler.mbn decode report, fuse-index mapping, license catalog
├── donor-analysis/          ← Donor XBL fuse-rule comparison table, CDSP status per device
└── device-state/            ← Full socinfo read, /proc/iomem grep, /dev listing, DT CDSP node
```

---

## What Is Included

All included evidence is **original analysis output or curated text**. Specifically:

- **On-device read outputs** — `cat /sys/devices/soc0/*`, `/proc/iomem`, `ls /dev/`, `getprop` dumps. Device serial `<SERIAL>` is replaced with `<SERIAL>` throughout; IMEI and `serial_number` fields are replaced with `<REDACTED>`.
- **`avbtool` info dumps** — partition descriptor output for `vbmeta_a`, `vendor_dlkm`, `boot`. No partition images are included; only the `avbtool` text output.
- **`e2fsck` / `dumpe2fs` output** — filesystem analysis from the brick postmortem. Partition images are not included.
- **XBL disassembly excerpts** — annotated snippets of the QFPROM fuse-read rule and NSP gate logic. The full XBL binary is not redistributed.
- **EDL session transcript** — Firehose protocol log (XML responses, NAK records). No Firehose programmer binary is included.
- **Build log excerpts** — `make` output showing successful module compilation, module list, patch diffs. No compiled `.ko` files are included.
- **FeatureEnabler decode report** — disassembly findings, fuse-index mapping table, license catalog. The `featenabler.mbn` binary is not redistributed.
- **Donor analysis table** — per-device CDSP status, `nsp`/`modem` values, XBL fuse-rule comparison. No donor firmware binaries are redistributed.
- **SHA-256 hashes** of firmware files — hashes are safe to publish; they allow independent verification that a given file matches what was analyzed here without requiring redistribution of the file.

---

## What Is Excluded

The following categories are **never committed** to this repository. They are documented here so readers understand what was used during the investigation but is not redistributable.

| Excluded Item | Reason |
|---------------|--------|
| AYN factory firmware binaries (`xbl_s`, `abl`, `tz`, `featenabler.mbn`, `boot`, `super`, `vbmeta`, `NON-HLOS`, `dspso`, …) | Proprietary AYN/Qualcomm copyright. Hashed and documented; binaries not published. |
| Donor firmware binaries (Realme GT7 Pro, Xiaomi 15 Ultra, iQOO 13, OnePlus 13) | Proprietary; used offline for comparison only. Referenced by source/hash, not redistributed. |
| Raw partition image dumps (`super_*.img`, `vendor_dlkm*.img`, `vbmeta*.img`, `boot*.img`, EDL backups) | Contain proprietary firmware and/or device-unique data. |
| Firehose programmer binaries (`prog_firehose*`, `*.melf`) | Signed Qualcomm/OEM tooling. |
| Compiled kernel modules, shared libraries, DTBs, APKs | Build outputs / proprietary blobs. |
| Device serial, IMEI, MEID, SIM identifiers | Privacy. Replaced with `<SERIAL>` / `<REDACTED>` in all published evidence. |

These exclusions are enforced by [`.gitignore`](https://github.com/mayusi/odin3-npu-teardown/blob/main/.gitignore).

---

## Evidence Subfolder → Documentation Mapping

| Evidence Subfolder | Primary Doc(s) That Cite It | What It Contains |
|-------------------|---------------------------|-----------------|
| `evidence/fuse-reads/` | [`02-how-we-proved-it/proof-1-socinfo.md`](../02-how-we-proved-it/proof-1-socinfo.md) · [`07-reproduce/index.md`](../07-reproduce/index.md) | `nsp=0xff`, `npu=0x0`, `modem=0xff` sysfs reads; `/proc/iomem` QFPROM window grep |
| `evidence/avb-verity/` | [`03-methods/07-vendor-dlkm-probes.md`](../03-methods/07-vendor-dlkm-probes.md) · [`03-methods/08-brick-and-recovery.md`](../03-methods/08-brick-and-recovery.md) · [`06-faq/index.md`](../06-faq/index.md) | `avbtool info` output for `vbmeta_a` and `vendor_dlkm`; before/after verity digest comparison (`d0f1abeb` → `d6572a40`) |
| `evidence/vendor-dlkm-postmortem/` | [`03-methods/08-brick-and-recovery.md`](../03-methods/08-brick-and-recovery.md) | `e2fsck`/`dumpe2fs` error output; dm-verity EIO trace; recovery session log; postmortem labels (`AVB_METADATA_BROKEN`, `EXT4_LAYOUT_BAD`) |
| `evidence/xbl-boot-policy/` | [`02-how-we-proved-it/proof-3-kernel-patent.md`](../02-how-we-proved-it/proof-3-kernel-patent.md) · [`03-methods/03-xbl-boot-policy.md`](../03-methods/03-xbl-boot-policy.md) | Annotated XBL disassembly excerpts: QFPROM read at `0x221c2420`, NSP gate logic, `feature_word[15]` derivation |
| `evidence/edl-firehose/` | [`03-methods/04-edl-firehose.md`](../03-methods/04-edl-firehose.md) · [`06-faq/index.md`](../06-faq/index.md) | EDL detection (USB VID/PID `0x05c6:0x9008`); Firehose session XML log; `peek` NAK response; backup partition manifest |
| `evidence/kernel-build/` | [`03-methods/06-kernel-build.md`](../03-methods/06-kernel-build.md) | CLO build log (`BUILD_READY_NO_FLASH`); compiled module list (`qcom_q6v5_pas.ko`, `qcom_fastrpc.ko`); log-only patch diffs |
| `evidence/feature-enabler/` | [`02-how-we-proved-it/proof-2-featenabler.md`](../02-how-we-proved-it/proof-2-featenabler.md) · [`03-methods/09-featenabler-decode.md`](../03-methods/09-featenabler-decode.md) | Trustlet decode report; `qsee_is_sw_fuse_blown(22)` = `NSP_DISABLE` mapping; AYN FeatureEnabler license catalog (display/video only) |
| `evidence/donor-analysis/` | [`03-methods/05-donor-firmware.md`](../03-methods/05-donor-firmware.md) | Per-device comparison table (4 × SM8750 donors): CDSP status, `nsp`/`modem` values, XBL fuse-rule presence |
| `evidence/device-state/` | [`02-how-we-proved-it/`](../02-how-we-proved-it/index.md) · [`07-reproduce/index.md`](../07-reproduce/index.md) | Full `soc0` sysfs dump; `/dev/fastrpc*` listing; CDSP DT node status; `getprop` excerpt (serial/IMEI redacted) |

---

## A Note on Completeness vs. Reproducibility

This evidence set is **curated, not exhaustive**. Raw investigation sessions produced much more output; only the artifacts directly relevant to the three proof paths are included. Every included file has been reviewed for sensitive data (serial numbers, personal paths) before publication.

The evidence is sufficient to independently verify every factual claim in the documentation without running any of the investigation steps yourself. For those who want to verify on their own Odin 3, the read-only reproduction guide at [`07-reproduce/index.md`](../07-reproduce/index.md) covers all checks that do not require specific firmware files.

For questions about what was excluded and why, see [SENSITIVE.md](https://github.com/mayusi/odin3-npu-teardown/blob/main/SENSITIVE.md).
