# Method 4: EDL / Firehose Forensics

**Date:** 2026-06-28
**Status:** EDL functional for read-only work; physical-memory `peek` not available with tested loader
**Evidence dirs:**
- `evidence/edl-firehose/` (proof summary, deep vetting report)
- `evidence/edl-firehose/LOADER_FUNCTIONS.md`

---

## Overview

Once the XBL analysis established that QFPROM address `0x221c2420` holds the NSP policy value and that this address is not reachable from Android userspace, the logical next step was Emergency Download Mode (EDL). EDL, also called Qualcomm Sahara/Firehose mode, gives a host PC a low-level channel to the device before Android starts. A Firehose loader with `peek` support can read arbitrary physical memory addresses — exactly what was needed to confirm the fuse value live.

This section documents what was attempted, what worked, what was confirmed read-only, and why the physical-memory peek remained out of reach.

---

## Loader Vetting

Before entering EDL, the loader binary and tooling were vetted offline. This was a deliberate safety step: uploading an untrusted MELF to a device in Qualcomm 9008 mode gives that code arbitrary hardware access.

### Loader: `iqoo_13.melf`

Three independent copies of the loader were compared:

| Source | Size | SHA-256 |
|---|---:|---|
| Staged local copy | 1,731,240 | `8CB97D1318D608A5D9FDD4AA6A888F9102CC902A84E1CE659500A7097E113E08` |
| Fresh download from Diablosat archive | 1,731,240 | `8CB97D1318D608A5D9FDD4AA6A888F9102CC902A84E1CE659500A7097E113E08` |
| Second fresh download (recheck) | 1,731,240 | `8CB97D1318D608A5D9FDD4AA6A888F9102CC902A84E1CE659500A7097E113E08` |

All three are byte-identical. The Temblast loader index entry (MD5 first-16: `9055e4b45ab5347c`) matches. Community references for Odin 3 EDL work (Gio Odin3tools guide, Pierre Carrier backup post) both cite `iqoo_13.melf` for SM8750 EDL.

Binary structure: a multi-ELF (MELF64) containing:
- `0x0` — ELF32, machine `0xF3` (RISC-V), entry `0x22126000`
- `0x904` — ELF32, machine `0xF3`, entry `0x20415100`
- `0x2E574` — ELF64, machine `0xB7` (AArch64), entry `0x14825FA8`
- `0x11F154` — ELF64, machine `0xB7`, entry `0x149997A8`
- `0x175704` — ELF64, machine `0x1`, entry `0x1494E000`

Relevant strings confirmed: Qualcomm/QCOM identifiers, UFS and `/soc/ufs0`, `sbl1_xblconfig_init`, Firehose XML command strings (`getstorageinfo`, `program`, `patch`, `erase`), Vivo device string `vivo_usb`, device-family string `PD2408`. A Windows Defender scan of staged artifacts returned no detections.

### Tool: `edl-ng` v1.5.0

| Artifact | SHA-256 |
|---|---|
| Release ZIP | `F66DE3E5962D0F31D4FEC18A02FDA5103400E56AB677FE38D2BD3F9EF5E79656` |
| `edl-ng.exe` | `9CEF98B45FAE0D89FA9ED6A411FD4166304F5A4C3D1DB5B72F16F2914751E96B` |
| `libusb-1.0.dll` | `72B0021906A2569B0F8E0FBCDB17DBDB37B88324787D9548617EE58629EF18AE` |

The release ZIP SHA-256 matches the GitHub release asset digest published at `https://github.com/strongtz/edl-ng/releases/tag/v1.5.0`. Build provenance: `.NET 9`, GitHub Actions workflow (`build.yml` + manual `release.yml` restricted to actor `strongtz`). No WinINet/WinHTTP imports, no covert persistence or auto-update signals observed in the PE import table.

The tool registers these sub-commands: `upload-loader`, `reset`, `printgpt`, `read-part`, `read-sector`, `dump-rawprogram`, `write-part`, `write-sector`, `erase-part`, `erase-sector`, `provision`, `rawprogram`. Destructive commands are present by design. Only read-only commands were approved for use in this run.

---

## EDL Session: Entering and Detecting

```
adb -s <SERIAL> reboot edl
```

The device entered Qualcomm 9008 mode. Windows enumerated it as:

```
Qualcomm HS-USB QDLoader 9008 (COM3)
```

Sahara handshake succeeded. Device serial reported by Sahara: `<SAHARA_SERIAL_REDACTED>`. The `iqoo_13.melf` loader was uploaded and started successfully. Firehose mode was detected post-loader.

---

## Read-Only Proof: GPT and Storage

The approved read-only surface was:

- `edl-ng printgpt --loader iqoo_13.melf --memory Ufs --lun N` (LUNs 0–7)
- `edl-ng reset` (to return to Android, outside initial proof boundary)

UFS storage information (LUN 0):

| Property | Value |
|---|---|
| Product | `HN8T374ZJKX141` |
| Vendor string | `SKhynix HN8T374ZJKX141 X202` |
| Block size | 4,096 bytes |
| Physical LUN count | 8 |
| Active LU bitmask | `0xff` |

GPT reads succeeded for LUNs 0–5. LUNs 6 and 7 returned `No GPT` (exit code 1), which is normal for empty or raw logical units.

**Partition map highlights:**

| LUN | Notable partitions |
|---|---|
| LUN 1 | `xbl_a`, `xbl_config_a`, `multiimgqti_a`, `multiimgoem_a`, `apdp` |
| LUN 2 | `xbl_b`, `xbl_config_b`, `multiimgqti_b`, `multiimgoem_b`, `apdpb` |
| LUN 3 | `cdt`, `ddr` |
| LUN 4 | `dsp_a/b`, `imagefv_a/b`, `abl_a/b`, `boot_a/b`, `dtbo_a/b`, `recovery_a/b`, `vbmeta_a/b`, `vendor_boot_a/b`, `init_boot_a/b` |
| LUN 5 | `modemst1`, `modemst2`, `fsg`, `fsc` |

Boot-critical image backups (`xbl_config_a`, `xbl_config_b`, `xbl_a`, `xbl_b`) were pulled read-only via `read-part` and used for the XBL offline analysis documented in Method 3.

---

## The `peek` Blocker

The core goal of the EDL session was to use Firehose physical-memory `peek` to read `0x221c2420` directly, confirming the fuse value live.

`edl-ng` does not expose a `peek` sub-command. The secondary tool used was `bkerler/edl` (community Python EDL client) which does implement `peek`. Two path attempts were made:

**Attempt 1 (streaming path):** The legacy Sahara/Streaming download protocol returned `b''` (empty bytes). This is not a meaningful read; the address was not reached.

**Attempt 2 (Firehose via `edl-ng` staging + bkerler):** After staging Firehose correctly with `edl-ng`, bkerler reached the Firehose protocol layer. The loader reported:

```
INFO: Chip serial num (0x0) Chip ID (0x0)
```

A local patch to `bkerler-edl-src/edlclient/Library/firehose.py` was applied to avoid crashing on this log format (not a device modification — a local script fix only). After patching, bkerler configured UFS storage successfully and executed `read` commands. Then:

```
Peek command isn't supported by edl loader
```

The Firehose supported-function list, as advertised by `iqoo_13.melf`, is:

```
program, read, nop, patch, configure, setbootablestoragedrive, erase, power,
firmwarewrite, getstorageinfo, benchmark, emmc, ufs, fixgpt, query_auth_id_info,
parse_sig, get_auth_id, program_backup_auth, getsha256digest, query_overlay_cfg,
makedp_info
```

There is no `peek`. There is no `poke`. The iQOO 13 Firehose loader is storage-oriented and does not implement physical-memory access. This is a capability gap in the loader, not a protocol error or configuration issue.

> See `evidence/edl-firehose/LOADER_FUNCTIONS.md` for the curated supported-functions reference.

---

## Result: What Was Proven, What Was Not

After all EDL sessions:

```
XBL rule:    feature_word[15] |= ((read32(0x221c2420) & 0xff) >> 0)
Android:     nsp = 0xff
QSPA:        ro.boot.vendor.qspa.nsp = disabled
```

The chain is confirmed end-to-end from static XBL analysis through Android runtime. The one gap:

```
EDL physical peek at 0x221c2420: NOT AVAILABLE with iqoo_13.melf
```

The live fuse value was never directly read. The circumstantial case is strong — the `0xff` result in `nsp` is consistent with the fuse having its NSP-disable bits set — but a direct Firehose peek would have closed the loop.

Post-session device state after `edl-ng reset`:

```
/sys/devices/soc0/nsp: 0xff
ro.boot.vendor.qspa.nsp: disabled
/dev/fastrpc-cdsp: absent
```

No change.

---

## Risk Assessment

> **Framing note:** This section describes actions that were taken and the risks that were evaluated, for transparency. It is not a recommendation to repeat these steps. Uploading any Firehose loader to a device in EDL mode is arbitrary code execution at a hardware-privilege level above the bootloader. Even a read-only session carries real risk of unintended side effects. The loader vetted here is a community-sourced third-party binary whose full behavior is not known.

| Risk | Assessment |
|---|---|
| Firehose loader provenance | Community-sourced; SHA-256 consistent across 3 sources; no malware signal |
| Side effects of loader upload | Cannot be fully excluded; loader initialises UFS as part of normal startup |
| Device survival | Device returned to Android after every session without incident |
| EDL re-entry | Confirmed available on all test sessions; device was not permanently wedged |
| Partition integrity | No write, erase, rawprogram, patch, or provision command was issued |

---

## What Would Be Needed for a Peek

A Firehose loader that advertises `peek` in its supported-function list and accepts physical addresses in the QFPROM corrected range (`0x221c2000–0x221c3fff`) would be needed. Such loaders exist (e.g. some engineering/test loaders for Snapdragon development platforms), but none were identified for SM8750 in the public loader corpus. Finding or obtaining one — if possible — would close the direct fuse-read gap.

---

## Summary

| Action | Outcome |
|---|---|
| Device enters EDL | Confirmed (Qualcomm 9008 on COM3) |
| Sahara handshake | Succeeded |
| `iqoo_13.melf` upload | Succeeded |
| UFS configuration | Succeeded |
| GPT reads (LUNs 0–5) | Succeeded |
| Boot-critical partition backup | Succeeded (read-only) |
| Device reset to Android | Succeeded |
| Physical-memory peek `0x221c2420` | **Not available** — loader does not support `peek` |
| Post-session device state change | None |
