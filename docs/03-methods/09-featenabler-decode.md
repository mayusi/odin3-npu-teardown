# 09 · FeatureEnabler Decode & Firehose Dead-End

> **Status:** `FEATENABLER_DOES_NOT_GATE_NSP` confirmed at binary level.
> The `_ns` Firehose programmer carries test-key signatures and does not register
> `peek` as a handler. Both avenues — FeatureEnabler as a soft provisioning path,
> and the Firehose programmer as a QFPROM peek tool — are closed.

---

## Why These Were Investigated

Two questions remained open after the XBL rule and donor analysis:

1. **FeatureEnabler:** Qualcomm's SoftSKU mechanism provisions features via
   RPMB-backed signed licenses. If the NPU were a licensable feature, an OEM
   could theoretically enable it by installing the right license. The
   `featenabler.mbn` partition was present in all donor firmwares. Was NSP one
   of its features?

2. **Firehose peek:** The only remaining path to reading `0x221c2420` (the
   QFPROM address XBL reads to gate NSP) directly was from EDL mode using a
   Firehose programmer with the `peek` verb. The Xiaomi 15 Ultra's
   `xbl_s_devprg_ns.melf` was the only loader that survived static screening
   with a `peek` string. What did its runtime analysis reveal?

---

## Part 1: FeatureEnabler (`featenabler.mbn`) Decode

### Provenance

The binary analyzed is from the **Realme GT 7 Pro (RMX5011)** donor extraction:

```
evidence/aime-npu-unlock-20260629/realme_gt7_pro_local_direct_entries/IMAGES__featenabler.mbn
MD5: 0d3fb501a7a5dcd9eb0cb3edfc64759d
Size: 105,440 bytes
```

A Xiaomi 15 Ultra sibling was also present. Both are SM8750 "sun" — the same
chipset family as the Odin 3. Qualcomm ships one `featenabler` per chipset BSP;
OEMs re-sign it. The conclusions transfer to the Odin's own `featenabler` with
high confidence.

**Note:** No AYN-provided `featenabler.mbn` was found in any Odin 3 firmware dump
on disk. The analysis is based on the donor proxy, which is the correct SM8750 BSP.

### Binary Identity

```
ELF64 / AArch64 / Type=DYN
Program headers: 9
Section headers: none (stripped)
Code segment: R-E, file offset 0x1000, vaddr 0x0, size 0xf0ff (~61 KB)
TA manifest: n=featenabler (B=true;n=featenabler;p=8:c47728cf3e4081,...)
Signing chain: OPLUS Attestation CA / OPLUS ROOT CA 1 / Qualcomm Cryptographic Operations
```

This is a signed **Qualcomm QTEE/TrustZone trustlet** — a Trusted Application (TA)
that runs in the secure world, not Linux. Strings confirm: *"feature license app
init/shutdown"*.

### What FeatureEnabler Actually Gates

The TA exposes exactly two feature cores. There is no third core and no
compute/DSP/NPU core of any kind:

**`DisplayCore_*`** — display feature gating:
```
DisplayCore_Open / Close / ConfigureSwFuse / EnableSwFuse / ReadSwFuseStatus
DisplayCore_GetFpsLimit / GetFpsLimitStatus / GetResolutionLimit / SetResFpsLimit
DisplayCore_GetFeatureList / GetFeatureName / EncodeMetaData
```
Feature names: `Display_ResFpsLimiter`, `Display_AntiAging`, `Display_Demura`,
`Display_SPR`, `Display_Allocate_Cache_Signal`, `FpsLimitIntf`, `HeightLimitIntf`,
`WidthLimitIntf`, `ResolutionLimitEnable`.

**`VideoCore_*`** — video resolution gating:
```
VideoCore_Open / Close / ConfigureSwFuse / EnableBit / ReadBitStatus
VideoCore_GetFeatureList / GetFeatureName / IsFeatureSupported / EncodeMetaData
```
Feature name: `Video_Resolution_Limiter`.

Everything else is plumbing: `IPFM_*` (In-Field Provisioning Manager), `RPMBUtils_*`,
`qsee_stor_read_sectors/write_sectors`, `HWIOUtils_Read/Write`, `SharedMemUtils_*`,
`CommonUtils_GetSocHwVersion`, full QCBOR codec (licenses are CBOR-encoded).

### NSP/NPU Token Search: Exhaustively Negative

Exhaustive string scan (`-n 3`) for all compute-related terms:

```
nsp  cdsp  hexagon  npu  turing  neural  tensor  dsp  compute
ai   ml    adsp     q6   hvx     aix     aip
```

**Result: zero hits.** Direct byte scan for `cdsp CDSP turing Turing hexagon
Hexagon Hvx HVX NSP q6v5 npu NPU neural Neural compute Compute` across the full
105 KB binary: **zero hits.**

**Verdict Q1: FeatureEnabler does not gate the NSP/CDSP/NPU/Hexagon. Its entire
feature catalog is consumer media limits — display FPS caps, display/video
resolution caps, and panel-quality features. The Hexagon compute subsystem is not
a FeatureEnabler-licensable feature on SM8750.**

Independent corroboration: Qualcomm names this component
`DisplayFeatureEnablerExecute` on the QCS8550 (same Hexagon-class IoT BSP) —
display-scoped, no compute role.

### Is `0x221c2420` a SwFuse FeatureEnabler Provisions?

**No.** Three independent checks:

1. **String scan:** No occurrence of `221c2420`, `221c2`, `0x221c`, or `qfprom`
   anywhere in the binary.
2. **Little-endian byte scan** for `20 24 1c 22` (= `0x221c2420`): not present.
3. **Full dword scan** of all 105 KB for any 32-bit value in the QFPROM aperture
   `0x221c0000–0x221cffff`: **0 matches.**

The address that XBL reads to gate NSP — `read32(0x221c2420) & 0xff` — does not
appear in FeatureEnabler's code or data at all. FeatureEnabler cannot be the
writer or provisioner of that value.

### What FeatureEnabler's "SwFuse" Actually Is

The `*_ConfigureSwFuse` / `EnableSwFuse` / `qsee_is_sw_fuse_blown` calls operate
on a software fuse that lives in **RPMB-backed secure storage**, gated by a
**signed CBOR license**, applied by writing a display/video HWIO register via
`HWIOUtils_Write`. The flow:

```
License (CBOR, signed)
  → installed in RPMB (qsee_stor_write_sectors)
  → FeatureEnabler validates (CFE_ValidateLicense, soc_hw_version check)
  → DisplayCore/VideoCore_ConfigureSwFuse
  → HWIOUtils_Write(<display/video MMIO>) + qsee_is_sw_fuse_blown
```

This is a different mechanism in a different address space from QFPROM `0x221c2420`.
"SwFuse" in FeatureEnabler = RPMB software-fuse + provisioned display/video MMIO
bit. The two systems are disjoint.

The disassembly confirms: no `movz/movk` pair in the 61 KB code segment builds any
`0x221c....` value. No `adrp` targets any `0x221c` page. The only MMIO the TA
touches is its own internal memory pages at `0xb000–0xe000`, `0x11000–0x12000`
(`.rodata`/`.data`). All display/video register access is mediated by the QTEE
`IPCore_OpenModule(module_id)` syscall — no hardcoded hardware addresses in user
TA code.

### Does Qualcomm License the NPU as a Feature?

**No public mechanism exists.** Qualcomm's SoftSKU / WES feature packs cover
display and media features exclusively. NPU/Hexagon never appears in that catalog.
Across Qualcomm's NPU product and developer documentation, the NPU is presented
as a fixed hardware engine whose TOPS varies by silicon tier and binning — not
by a software license that can be installed.

On the **Dragonwing Q-8750** (the AIoT sibling of CQ8725S/CQ8750S), the Hexagon
V79 NPU at 77 dense TOPS is the headline feature — the AI engine is what the
product exists to sell, not something gated behind a license.

CQ8725S's documented IoT-binning removals are modem-RF and camera/ISP, not the NPU.

### Effect on Feasibility Estimate

FeatureEnabler was the last plausible "soft, provisionable" unlock path. Its
elimination pushes the NSP-disable verdict from approximately 0.70-permanent toward
**~0.78–0.80-permanent**. The one signed, RPMB-provisionable feature-licensing
mechanism in this firmware has no NSP/CDSP feature and makes zero reference to
`0x221c2420`. The NSP disable lives on a separate XBL-read fuse path that
FeatureEnabler cannot touch.

---

## Part 2: Firehose Programmer (`xbl_s_devprg_ns.melf`) Analysis

### File Identity

```
Size:       1,704,356 bytes (1.70 MB)
Build date: Oct 31 2025, 14:54:43
Format:     Qualcomm MELF (Multi-ELF) container
```

The MELF contains four nested images:

| Offset | Type | Contents |
|--------|------|----------|
| `0x000000` | ELF32 (outer stub) | Bootstrap wrapper, 2 tiny LOAD segments |
| `0x000904` | ELF32 (12 phdrs) | **The actual devprg (Firehose) programmer** |
| `0x02E574` | ELF64 AArch64 | XBL_CORE secondary boot image |
| `0x124154` | ELF64 AArch64 | XBL sub-image (contains peek strings in rodata) |
| `0x17014C` | ELF64 AArch64 | Additional XBL sub-image |

The Firehose programmer is the ELF32 at `0x904`. Its verb dispatch loop is the
only part that matters for the peek question.

### Registered Verb Handlers

Exact null-separated verb table at file offset `0x161523`:

```
program  read  nop  patch  configure  setbootablestoragedrive  erase
power  firmwarewrite  getstorageinfo  benchmark  emmc  ufs  fixgpt
getsha256digest
```

**`peek` is not registered. `poke` is not registered.**

The string `"size in bytes is %d, nothing to peek/poke\0"` exists at `0x161669`
but it is embedded inside an error string, not a handler registration. Sending
`<peek address64="0x221c2420" SizeInBytes="16"/>` to this programmer hits the
"Failed to read XML command" fallback — there is no peek dispatcher.

### The Peek Dead Code

The peek *implementation* exists — in the XBL sub-image at `0x124154` (ELF3,
vbase `0x14999000`), not in the devprg ELF. The strings are there:

```
"Peek is disabled on secure boot devices"   @ binary offset 0x1613D7
"Using address %p"
"0x%02X "
"Error while copy data to output buffer %d"
"address64"
```

But this XBL sub-image is never reached via the devprg XML dispatch loop. The
peek code is dead code from the perspective of any Firehose client.

The gate string is explicit: **"Peek is disabled on secure boot devices."** This
is a one-way check against live hardware secure-boot state. On a retail Odin 3
with production fuses, this check returns the disabled message and returns
immediately — even if the verb were registered.

### Signing Chain: Test Keys

The inner devprg ELF hash segment (PHDR11, file offset `0x2C208`, size `0x2C70`)
carries this certificate chain:

| Position | Issuer |
|----------|--------|
| Leaf | Qualcomm Technologies — SRoT MBNv7 Image Signing Root CA 6 SubCA 1 (EC secp384r1) |
| SubCA | **`SECTOOLS SECP384R1 CURVE TEST ROOT0` / "General Use Test Key 0 (for testing only)"** |
| Root-1 | **`SECTOOLS SECP384R1 CURVE TEST ROOT` / "General Use Test Key (for testing only)"** |
| Root-2 | **`SECTOOLS SECP384R1 CURVE TEST ROOT` / "General Use Test Key (for testing only)"** |

The OEM string: `OEM_IMAGE_UUID_STRING=Q_SENTINEL_{3783DB2D-0392-40D0-878B-347C2072EA49}_20250423_0600`

This programmer is signed with **Qualcomm SECTOOLS test/development keys**. The
Odin 3's PBL verifies the programmer's cert chain against the production OEM key
hash burned in fuses. A test-signed programmer will be **rejected by the production
PBL** unless the device is in engineering/test mode.

### Summary Table

| Question | Answer |
|----------|--------|
| `peek` registered as Firehose handler? | **NO** — not in verb table |
| `poke` registered as Firehose handler? | **NO** |
| Peek implementation present in binary? | Yes — dead code in XBL sub-image (ELF3) |
| Peek runtime gate | `secure_boot_fused == false` — always disabled on retail Odin |
| Programmer signed with OEM production key? | **NO** — SECTOOLS test keys |
| Will load on retail Odin 3 PBL? | **NO** — rejected by production fuse/cert chain |
| VIP mechanism that bypasses peek gate? | No — VIP authenticates write payloads, not peek |

The Firehose peek avenue is closed at two independent levels:
1. The programmer will be rejected by the PBL before it can execute anything.
2. Even if it loaded, `peek` is not a registered verb.

---

## What Would Actually Work

To read `0x221c2420` and confirm whether the NSP disable is a blown OTP cell or
a re-provisionable row:

1. A production-signed Firehose programmer from AYN or Qualcomm, with `peek`
   registered, on a device where the test key condition is met — not publicly
   available.
2. A kernel driver reading `/dev/mem` or via SMMU-bypassed iomem on a rooted
   device — would require kernel modifications and the same DT/boot-policy
   barriers as [Method 06](06-kernel-build.md).
3. JTAG/cJTAG on a test device or engineering-fused unit.

None of these paths are available without AYN/Qualcomm cooperation.

---

## Combined Verdict

| Question | Answer | Confidence |
|----------|--------|------------|
| Does FeatureEnabler list/gate NSP/CDSP/NPU? | **NO** | Very High |
| Is `0x221c2420` a SwFuse FeatureEnabler provisions? | **NO** — independent XBL fuse read | High |
| Does Qualcomm license the NPU as a software feature (CQ/Dragonwing)? | **No public mechanism** | Medium-High |
| Does `xbl_s_devprg_ns.melf` load on retail Odin 3 PBL? | **NO** — test-signed | High |
| Does the programmer expose `peek`? | **NO** — not a registered handler | Certain |
| Net effect on unlock feasibility | Permanent probability increases to ~0.78–0.80 | — |

The FeatureEnabler and Firehose investigations close the last two "maybe software
can fix this" avenues. The NSP disable is now best characterized as a hardware
fuse state consumed by XBL on a code path that no available software tool can
reach on a retail, production-fused device.

> **Evidence files:**
> `evidence/aime-npu-feasibility-20260630/FEATENABLER_DECODE.md`
> `evidence/aime-npu-feasibility-20260630/DEVPRG_NS_PEEK_ANALYSIS.md`
> `evidence/aime-npu-unlock-20260629/realme_gt7_pro_local_direct_entries/IMAGES__featenabler.mbn` (analyzed binary)
