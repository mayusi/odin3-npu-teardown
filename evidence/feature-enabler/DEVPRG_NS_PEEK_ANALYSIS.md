# DEVPRG_NS_PEEK_ANALYSIS.md
## `xbl_s_devprg_ns.melf` — Offline Firehose Programmer Analysis
**Date:** 2026-06-30  
**File:** `<home>\Downloads\ODIN3_FOLDERS\xbl_s_devprg_ns.melf`  
**Size:** 1,704,356 bytes (1.70 MB)  
**Build date (embedded):** Oct 31 2025, 14:54:43

---

## 1. Structure: What this MELF actually is

The file is a Qualcomm MELF (Multi-ELF) container with four nested images:

| Offset | Type | Description |
|--------|------|-------------|
| `0x000000` | ELF32 (outer wrapper) | 2 tiny LOAD segs (0x600 + 0x280 bytes) — bootstrap stub |
| `0x000904` | ELF32 (inner, 12 phdrs) | **The actual devprg (Firehose) programmer** — RISC-V/Thumb2, loads to 0x20415100 |
| `0x02E574` | ELF64 AArch64 | XBL_CORE (secondary boot image) |
| `0x124154` | ELF64 AArch64 | Additional XBL sub-image (contains devprg rodata + peek strings) |
| `0x17014C` | ELF64 AArch64 | Additional XBL sub-image |

The devprg (Firehose) programmer is the ELF32 at `0x904`. Its code segment (PHDR5) is at file `0x1A04–0x22B9C`, and its rodata (verb table + error strings) is distributed across the other segments.

---

## 2. Registered Firehose XML Verb Handlers (Supported Functions)

**Exact null-separated verb table found at file offset `0x161523`:**

```
program
read
nop
patch
configure
setbootablestoragedrive
erase
power
firmwarewrite
getstorageinfo
benchmark
emmc
ufs
fixgpt
getsha256digest
```

**`peek` and `poke` are NOT registered handlers.**

There is no null-terminated `peek\0` in the verb table. A null-terminated `poke\0` exists at `0x161669` but is embedded inside the error string `"size in bytes is %d, nothing to peek/poke\0"` — not a handler registration.

The "Supported Functions (%d):" / "End of supported functions %d" printf calls enumerate only the 15 verbs above. If a Firehose client sends `<peek address64="..." SizeInBytes="..."/>`, the dispatcher will hit the "Failed to read XML command" fallback — there is no peek handler to invoke.

---

## 3. Peek/Poke Code: Dead Code Path, Not an Active Handler

The programmer contains these strings that suggest a peek/poke *implementation* exists in the binary:

```
"Peek is disabled on secure boot devices"    @ binary offset 0x1613D7 (ELF3 virt 0x149D5283)
"Using address %p"
"0x%02X "
"Error while copy data to output buffer %d"
"size in bytes is %d, nothing to peek/poke"
"address64"
"Invalid address of NULL %d"
"Can't decode attribute %s with value %s"
```

These strings reside in ELF3 (the third sub-image, vbase 0x14999000) — NOT in the devprg ELF. This sub-image is an XBL component that has its own separate Firehose-like handler, but this code is never reached via the devprg's XML dispatch loop because `peek` and `poke` are absent from the devprg verb table.

**The peek gate, verbatim:**

> `"Peek is disabled on secure boot devices"`

This is a one-way check: if the device has secure boot fused (i.e., the PBL enforces authentication), the peek path prints this string and returns without reading. The condition is checked against live hardware state — it is NOT a VIP/signed-table bypass.

Additional relevant strings confirm the authentication model:
```
"Requesting VIP img authentication elf_buf = 0x%x, elf_buf_len = %d"
"VIP is enabled, receiving the signed table of size %d"
"Authentication of signed hash failed %d"
```
VIP (Verified Image Protocol) is for **program/write** operations — it authenticates data being flashed, not peek access. There is no VIP path that enables peek on a secure-boot device.

**Conclusion on gate:** Peek is gated by the secure-boot fuse state, checked at runtime. On a retail AYN Odin 3 (which ships with secure boot enabled), the peek path would print the disabled message and return — even if peek were a registered handler. It is not registered, which is the earlier blocking point.

---

## 4. Signing / OEM Certificate Analysis

The inner devprg ELF (at `0x904`) has a **hash segment** (PHDR11, type=0 flags=`0x02000000`) at file offset `0x2C208`, size `0x2C70` (11,376 bytes).

The hash segment begins immediately with DER X.509 data (no leading SHA256 entries), indicating the cert chain is at the front of the combined hash+cert+sig block.

**Cert chain issuers extracted from hash segment:**

| Cert | Issuer / Subject |
|------|-----------------|
| Leaf (hash+0x00CA) | **Qualcomm Technologies, Inc. — SRoT MBNv7 Image Signing Root CA 6 SubCA 1** — EC secp384r1, valid 200825–500825 |
| SubCA (hash+0x092C) | `SECTOOLS SECP384R1 CURVE TEST ROOT0` / `General Use Test Key 0 (for testing only)` — valid 251031–451026 |
| Root-1 (hash+0x0BC5) | `SECTOOLS SECP384R1 CURVE TEST ROOT` / `General Use Test Key (for testing only)` — valid 160321–360316 |
| Root-2 (hash+0x0EB9) | `SECTOOLS SECP384R1 CURVE TEST ROOT` / `General Use Test Key (for testing only)` — valid 160321–360316 |

**Critical finding: this programmer is signed with SECTOOLS TEST/DEVELOPMENT keys, NOT production OEM keys.**

The leaf cert was issued by Qualcomm's SRoT MBNv7 production root, but the intermediate chain contains `"General Use Test Key (for testing only)"` and `SECTOOLS TEST ROOT` issuers. This is a **test-signed** programmer.

The OEM string `OEM_IMAGE_UUID_STRING=Q_SENTINEL_{3783DB2D-0392-40D0-878B-347C2072EA49}_20250423_0600` confirms this was built for internal QC/OEM development use. The Odin 3's PBL checks the OEM fuse key hash against the programmer's cert chain. A test-signed image will be **rejected by a production-fused PBL** unless the device is in engineering/test mode.

---

## 5. Exact Read-Only Command Sequence

If this programmer were to work, the Firehose XML for a physical memory peek would be:

```xml
<?xml version="1.0" ?>
<data>
  <peek address64="0x221c2420" SizeInBytes="16" />
</data>
```

**edl-ng (bkerler) equivalent:**
```
edl --loader xbl_s_devprg_ns.melf peek 0x221c2420 16
```

**HOWEVER:** this command will fail at the dispatcher — `peek` is not a registered handler in this programmer. The response will be the "Failed to read XML command" error.

**DO NOT USE:**
- `<poke ...>` — not registered, would fail; even if it worked, it writes physical memory (DANGEROUS)
- `Static Fuse Overwrite` — would permanently blow fuses (IRREVERSIBLE, device-bricking)
- `<program ...>` / `<erase ...>` — flash write operations, potentially destructive

---

## 6. Risk Assessment and Honest Verdict

### Will `xbl_s_devprg_ns.melf` load on the Odin 3 in EDL?

**Almost certainly NO.** The programmer is signed with Qualcomm SECTOOLS test keys. The Odin 3 is a retail device with production OEM fuses. The PBL will verify the hash segment signature against the burned OEM key hash and reject a test-signed programmer.

### Even if it loaded, would peek work?

**NO.** The registered verb table contains 15 handlers. `peek` is not one of them. Sending a `<peek>` XML command would hit the unknown-command fallback.

### Does the binary contain peek implementation?

**YES, as dead code** in a separate XBL sub-image (ELF3). The implementation exists and is functional, but it is not exposed through the devprg Firehose XML dispatch. Even if it were reachable, the explicit `"Peek is disabled on secure boot devices"` gate would block it on a retail device.

### Summary

| Question | Answer |
|----------|--------|
| `peek` registered as Firehose handler? | **NO** |
| `poke` registered as Firehose handler? | **NO** |
| Peek implementation present in binary? | Yes (dead code in ELF3 sub-image) |
| Peek gate condition | `secure_boot_fused == false` — retail Odin = always disabled |
| Programmer OEM-signed (production key)? | **NO** — test-signed (SECTOOLS test keys) |
| Likely to load on retail Odin 3 PBL? | **NO** — will be rejected |
| VIP bypass to enable peek? | No — VIP is for write auth, not peek unlock |
| READ-ONLY safety of peek (if it worked) | Peek/getstorageinfo are read-only; poke/program/erase are NOT |

### Bottom line

The `xbl_s_devprg_ns.melf` programmer does **NOT** expose peek/poke as Firehose XML handlers and is **NOT** signed with production OEM keys. It cannot read 0x221C2420. The path remains blocked. To read 0x221C2420, you need either:
1. A production-signed programmer (from AYN/Qualcomm, not publicly available) that has `peek` registered as a handler AND the device not in secure-boot mode, OR
2. A kernel driver approach (kernel module reading `/dev/mem` or via SMMU-bypassed iomem access) on a rooted device, OR
3. A SoC-specific side-channel or debug register if exposed (e.g., JTAG/cJTAG on a test device).
