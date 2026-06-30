# Method 3: XBL Boot Policy Analysis

**Date:** 2026-06-28
**Status:** Policy traced to QFPROM `0x221c2420`; XBL is SecBoot SHA-384 sealed and cannot be patched without a Qualcomm signing chain
**Evidence dirs:**
- `evidence/xbl-boot-policy/` (xbl\_config offline analysis, patched candidate, enforcement report)
- `evidence/donor-analysis/` (ADDRESS\_ATTRIBUTION\_0x221c2420)

---

## The Central Question

The firmware inventory established that CDSP is excluded from `RetailImages` in `xbl_config`, and that `/sys/devices/soc0/nsp = 0xff` is the runtime signal of that exclusion. This page documents the investigation into **why** `nsp` is `0xff` and whether any accessible lever could change it.

The answer leads to a hardware fuse read buried inside XBL.

---

## The XBL NSP Rule

Static analysis of both the Odin 3 `xbl_a` image and a donor SM8750 image (Realme GT 7 Pro, also SM8750 / "sun") reveals a common rule cluster:

```
srcPtr = 0x221c2420
mask   = 0xff
part   = 15         ← socinfo part 15 = nsp
```

XBL reads a 32-bit value from physical address `0x221c2420`, masks the low 8 bits, and publishes the result as socinfo part 15 (the NSP capability field). In pseudocode:

```c
feature_word[15] |= (read32(0x221c2420) & 0xff);
```

This rule was found at:

| Image | Offset |
|---|---|
| Odin 3 `xbl_a` | `0x109354` |
| Realme GT 7 Pro `xbl_a` (donor) | `0x103f54` |

The fact that both devices share the same rule at similar offsets confirms this is platform-common Qualcomm XBL code for SM8750, not a device-specific AYN customisation.

---

## Address Attribution: `0x221c2420`

The physical address `0x221c2420` was cross-referenced against:

- The kernel QFPROM binding documentation
- SA8775P LKML discussions covering the same `0x221c....` block
- The Odin 3 live device tree

The address space model:

| Range | Description |
|---|---|
| `0x221c0000 – 0x221c1fff` | Raw QFPROM read |
| `0x221c2000 – 0x221c3fff` | ECC-corrected QFPROM read |
| `0x221c8000 – 0x221c8fff` | HLOS DT-exposed qfprom window on Odin 3 |

`0x221c2420` falls inside the **ECC-corrected QFPROM range** — the read side of a hardware fuse block. This is not a configurable register, a shadow latch, or an OTP that can be set after manufacture by normal means. It is a fused policy bit.

The Odin 3 DT exposes only `qfprom@221c8000` (size `0x1000`) to Android HLOS:

```
/sys/firmware/devicetree/base/soc/qfprom@221c8000/reg   = 221c8000 00001000
/sys/firmware/devicetree/base/soc/qfprom@221c8000/compatible = qcom,sun-qfprom; qcom,qfprom
```

Android's visible nvmem window (`qfprom12`, 4096 bytes) starts at `0x221c8000` and does not cover `0x221c2420`. There is no userspace path to read that fuse from Android.

**Classification:** `QFPROM_CORRECTED_RANGE_POLICY_INPUT_FOUND`

The value at `0x221c2420` is best understood as a per-unit fuse (or fuse row) programmed at chip manufacture or at AYN's ODM personalisation stage. On this Odin 3, the low byte reads as `0xff`, which maps to "all NSP cores disabled."

---

## `xbl_config`: The `RetailImages` Gate

XBL uses `xbl_config` to determine which images to authenticate and load during secure boot. The stock `RetailImages` entry for Odin 3:

```
ABL, SPSS, ImageFv, FULL_ADSP, FULL_ADSP_DTB
```

`FULL_CDSP` and `FULL_CDSP_DTB` are absent.

However, the CDSP config *nodes* already exist inside `xbl_config`:

```
FULL_CDSP_CFG        offset 0x12134
FULL_CDSP_DTB_CFG    offset 0x11f58
CORE_CDSP_CFG        offset 0x13038
CORE_CDSP_DTB_CFG    offset 0x12e50
```

And in `xbl_a`, NSP voltage-rail references appear at `0xc8898` and `0xc889c`, and CDSP DT-related config strings exist at multiple offsets. The hardware and software infrastructure for CDSP is present throughout the boot chain. Only the `RetailImages` list and the QFPROM fuse prevent it from activating.

This creates two candidate intervention points:

1. **Patch `xbl_config`** — add `FULL_CDSP` and `FULL_CDSP_DTB` to `RetailImages`
2. **Override the QFPROM read** — somehow change or bypass the `0x221c2420` read in XBL

Both are explored below, and both are blocked.

---

## Candidate 1: Patching `xbl_config`

`xbl_config` is an ELF/FDT image with a MBNv7 authentication region. The partition is 352,256 bytes on this device.

### Stock image hashes

```
xbl_config_a SHA256: 94c07b0e4c71af8f9b5a5492e7b242ba9607a65b692189d17051ebc2e00c5102
xbl_config_b SHA256: 94c07b0e4c71af8f9b5a5492e7b242ba9607a65b692189d17051ebc2e00c5102
```

Both slots are byte-identical.

### The auth structure

The MBN authentication region is PHDR `13` at offset `0x52000`, size `0x1148`. It contains SHA-384 hashes for each of the 13 loadable program headers (PHDR 0–12).

```
Auth segment SHA256: 755c5491aee259c5816dbfac31e541dccf40b217418aa739dc2618f725739557
MBN header version: 7
```

Strings in `xbl_a` confirm active segment hash checking:
- `boot_elf_auth.c` (at `0xc4f92`)
- `Auth Metadata` (at `0xc5182`)
- `Segments hash check` (at `0xc5190`)
- `Secure Boot:` (at `0xc0677`)
- `not authorized` (at `0xc7b95`)

### Additive patch (appending FULL_CDSP to RetailImages)

An additive patch was generated offline and analysed. It adds `FULL_CDSP` and `FULL_CDSP_DTB` to the `RetailImages` FDT node inside PHDR 3:

```
Patched SHA256: 776c2d5080d4f4c3542fd9970e74972d0469eb550e0dbf94052c1cb5ee60a0b5
```

The patch is structurally valid ELF/FDT, but it breaks authentication in two places:

| PHDR | Stock hash in auth region | Patched hash in auth region |
|---|---|---|
| 0 (program headers) | Found at `0x52120` | **Not found** — old stock hash remains |
| 3 (main FDT) | Found at `0x521b0`, prefix `83a13b429c491d5e` | **Not found** — new hash prefix `56ffaa5e535e3125` |

PHDR 0 changes because adding bytes to the FDT segment shifts the program header table. PHDR 3 changes because its content changes. The auth region is not updated (it is hash-covered by the ELF/MBN outer layer). The auth segment SHA256 is the same between stock and patched (`755c5491...`), meaning the outer signature has not changed — but the per-segment hashes it covers no longer match the patched content.

### Same-size patch (FULL\_ADSP → FULL\_CDSP byte swap)

A same-size variant was also explored — replacing `FULL_ADSP` with `FULL_CDSP` byte-for-byte at offsets `0x11719` and `0x11723`, preserving file size and ELF layout:

```
Same-size patched SHA256: 24707a5dd080c4e4b3cd19e98b40fd2764902165ac24d997a11eac3f9d56be6d
```

Result: avoids growing the FDT or shifting the SecTools block, but PHDR 3 content still changes, so its SHA-384 still does not match the stored hash. Additionally, this removes ADSP from `RetailImages`, which would likely break audio and other ADSP-dependent subsystems.

### Why these patches cannot be flashed safely

XBL verifies `xbl_config` segment hashes before using any data from it. If PHDR 3's hash does not match the stored value, XBL will reject the image. The consequence — depending on whether the rejection is a hard abort or a fallback — is either a boot loop or silent use of stock config, either of which defeats the patch's purpose.

For the patch to be accepted, the auth region would need to be updated with new SHA-384 hashes, and the outer ELF/MBN signature covering the auth region would need to be re-signed. That signing chain requires the Qualcomm/OEM production keys — `SECP384R1 CURVE TEST ROOT` is the signing curve embedded in `xbl_config`, and the only keys that can produce a valid signature are those held by Qualcomm or AYN.

The local Qualcomm SDK (`sectools`) was checked and confirmed inapplicable: it provides only a `tcg` chipset profile and RSA test keys, not the SM8750/MBNv7/P-384 signing chain needed here.

---

## Candidate 2: Bypassing the QFPROM Read in XBL

The `0x221c2420` read happens in XBL itself, before any userspace or Android-layer software runs. The only way to intercept it would be to patch XBL — the code that implements the fuse read.

`xbl_a` is a 3,670,016-byte ELF64 image:

```
xbl_a SHA256: c206ab6e35b0987b146d0b664d799fb7b3a1d7a2fa8bc29bb9542c643d183264
xbl_b SHA256: c206ab6e35b0987b146d0b664d799fb7b3a1d7a2fa8bc29bb9542c643d183264
```

Both slots are byte-identical. XBL is Secure Boot sealed with a SHA-384 hash-per-segment chain under an OEM production key (SECP384R1 curve, same as `xbl_config`). The `SECTOOLS SECP384R1 CURVE TEST ROOT` chain embedded in the auth region of `xbl_a` is the *test* chain; the production device enforces a production key. Any binary modification to `xbl_a` will fail signature verification during Secure Boot.

`ro.boot.verifiedbootstate = orange` (unlocked bootloader), but unlocked bootloader on SM8750 means Android Verified Boot is relaxed — it does not mean Qualcomm Secure Boot for XBL/ABL is disabled. Qualcomm Secure Boot (eFuse SHA-384 hash chain) operates independently of AVB and cannot be bypassed by bootloader unlock.

---

## What Was Not Achievable

| Approach | Outcome |
|---|---|
| Read `0x221c2420` from Android userspace | HLOS nvmem window starts at `0x221c8000`; does not cover the address |
| Read `0x221c2420` via EDL Firehose `peek` | `peek` not supported by `iqoo_13.melf` — see Method 4 |
| Patch `xbl_config` to include `FULL_CDSP` | Breaks PHDR 3 and PHDR 0 SHA-384 in auth region; unsigned |
| Patch `xbl_a` to NOP the fuse read | XBL is SecBoot SHA-384 sealed; any binary change is rejected |
| Find a donor `xbl_config` with CDSP in `RetailImages` | Not located; would still need to be for the correct SM8750/AYN signing identity |
| OEM/AYN official firmware with CDSP enabled | Not released as of 2026-06-28 |

---

## Summary

The NSP/CDSP block on the AYN Odin 3 traces to a single hardware fuse at physical address `0x221c2420` in the ECC-corrected QFPROM range. XBL reads this fuse, sets `nsp = 0xff` in socinfo, and the QSPA service propagates that as `ro.boot.vendor.qspa.nsp = disabled`. The `xbl_config` `RetailImages` omission is a consistent companion policy — it would also need to be changed — but the fuse read in XBL is the root cause.

Both the XBL binary and the `xbl_config` image are SHA-384 signed under Qualcomm/OEM production keys. Neither can be patched and re-signed without that key material. The hardware fuse itself cannot be read from Android and cannot be written after chip manufacture.

The investigation reached the end of the accessible surface. For EDL attempts to read the fuse directly, see [Method 4: EDL / Firehose Forensics](./04-edl-firehose.md).
