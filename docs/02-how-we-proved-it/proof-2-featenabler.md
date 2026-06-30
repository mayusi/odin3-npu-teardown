# Proof 2 — FeatureEnabler Trustlet Disassembly

**Method:** Static binary analysis. No device, adb, fastboot, or EDL access. Offline only.

**Binary analyzed:** `featenabler.mbn` from Realme GT7 Pro (RMX5011) OTA — an SM8750 "sun" donor.
MD5: `0d3fb501a7a5dcd9eb0cb3edfc64759d` · Size: 105,440 bytes
Qualcomm ships one `featenabler` per chipset BSP; OEMs re-sign the same binary. The SM8750 donor is a valid proxy for the Odin 3's equivalent trustlet.

**Evidence files:**
- `evidence/feature-enabler/FEATENABLER_DECODE.md`

---

## What Is FeatureEnabler?

`featenabler.mbn` is a signed **QTEE/TrustZone trustlet** — an AArch64 ELF binary that runs inside Qualcomm's secure execution environment, outside the reach of the Android OS. Its role is to validate signed feature licenses (stored in RPMB) and apply feature gates on deployed hardware.

Binary identity:
```
Format:   ELF64 / AArch64 / Type=DYN
Segments: 9 program headers; no section headers
Code:     R-E segment at file offset 0x1000, vaddr 0x0, size 0xf0ff (~61 KB)
Signer:   OPLUS Attestation CA / OPLUS ROOT CA 1 / Qualcomm Cryptographic Operations
TA name:  featenabler (manifest: B=true;n=featenabler)
Self-ID:  "feature license app init/shutdown"
```

---

## The NSP-Disable Fuse Check

Within the trustlet's code, a call to `qsee_is_sw_fuse_blown` appears with fuse ID 22:

```asm
; offset 0x76b0 in code segment
76b0:  mov  w0, #0x16          ; fuse_id = 22 (0x16)
76b4:  add  x1, sp, #4         ; &out_blown_status
76b8:  mov  w2, #1
76cc:  bl   qsee_is_sw_fuse_blown
```

Fuse ID 22 maps to `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]` — a **dedicated hardware OTP fuse** in the QFPROM feature-configuration row. The `qsee_is_sw_fuse_blown` syscall is a TrustZone API that reads fuse state; when the NSP_DISABLE bit is blown (factory-set to 1), it returns "blown."

This call is **purely a read**. The trustlet checks the fuse but cannot set it — QFPROM OTP cells are write-once at manufacturing and are physically impossible to reset.

---

## What FeatureEnabler Actually Licenses

The trustlet exposes exactly **two feature domains**. The complete feature surface recovered from strings and handler table analysis:

### DisplayCore — Display feature gating

```
Handlers:
  DisplayCore_Open / Close
  DisplayCore_ConfigureSwFuse / EnableSwFuse / ReadSwFuseStatus / GetSwFuseStatus
  DisplayCore_GetFeatureList / GetFeatureName / GetFeatureString / IsFeatureSupported
  DisplayCore_GetFpsLimit / GetFpsLimitStatus / GetResolutionLimit / GetResolutionLimitStatus
  DisplayCore_SetResFpsLimit / EncodeMetaData

Feature names (from .rodata):
  Display_ResFpsLimiter
  Display_AntiAging
  Display_Demura
  Display_SPR
  Display_Allocate_Cache_Signal
  FpsLimitIntf  /  HeightLimitIntf  /  WidthLimitIntf  /  ResolutionLimitEnable
```

### VideoCore — Video resolution gating

```
Handlers:
  VideoCore_Open / Close
  VideoCore_ConfigureSwFuse / EnableBit / ReadBitStatus / GetSwFuseStatus
  VideoCore_GetFeatureList / GetFeatureName / IsFeatureSupported
  VideoCore_FillCommonLicenseInfo / EncodeMetaData

Feature names (from .rodata):
  Video_Resolution_Limiter
  WidthLimitIntf
```

### NSP/NPU/CDSP token search — exhaustively negative

A full string grep across the binary for every relevant identifier:

```
Search terms: nsp | cdsp | hexagon | npu | turing | neural | tensor | dsp |
              compute | ai | ml | adsp | q6 | hvx | aix | aip
Result: zero hits (only generic false-positives from unrelated strings)

Direct byte scan for substrings:
  cdsp CDSP turing Turing hexagon Hexagon Hvx HVX NSP q6v5 npu NPU
  neural Neural compute Compute
Result: zero matches
```

**The NSP/CDSP/NPU/Hexagon compute subsystem does not appear anywhere in FeatureEnabler.** Its entire catalog is consumer media limits: display FPS cap, display and video resolution limits, and panel-quality features (Demura, SPR, AntiAging).

---

## The SwFuse Mechanism — What It Is and Is Not

FeatureEnabler uses a mechanism called "SwFuse" (`ConfigureSwFuse`, `EnableSwFuse`, `ReadSwFuseStatus`). This is **not the same thing** as the QFPROM OTP fuse at `0x221c2420`. The two mechanisms are completely disjoint:

### FeatureEnabler's SwFuse (display/video only)

```
Signed CBOR license → installed in RPMB (qsee_stor_write_sectors)
     ↓
FeatureEnabler validates (CFE_ValidateLicense + soc_hw_version check)
     ↓
DisplayCore/VideoCore_ConfigureSwFuse
     ↓
HWIOUtils_Write(<display or video MMIO register>)
+ qsee_is_sw_fuse_blown check
```

The SwFuse is an **RPMB-backed software license** whose enforcement is applied by writing a runtime-mapped display/video MMIO register via the QTEE `IPCore_OpenModule` dispatch. It lives in provisioned secure storage and is the mechanism Qualcomm calls "SoftSKU" or "WES feature pack" — it can be installed or upgraded on deployed hardware.

### The NSP fuse at `0x221c2420` (hardware OTP)

```
Physical QFPROM OTP cell at 0x221c2420
     ↓
XBL reads at boot (full QFPROM aperture access)
     ↓
Decoded as socinfo nsp byte → published to SMEM
     ↓
Kernel: /sys/devices/soc0/nsp = 0xff
```

This lives in the hardware fuse array. Once blown at the factory, it cannot be reset. XBL reads it very early in boot, before any trustlet (including FeatureEnabler) is loaded.

### The decisive negative: FeatureEnabler never touches `0x221c2420`

A comprehensive scan of the entire binary for any reference to the QFPROM aperture:

```
String scan:     no occurrence of 221c2420, 221c2, 221c3, 221c0, 221c8, 0x221c, qfprom
Little-endian byte scan for 0x221c2420 (bytes 20 24 1c 22): not present
Full dword scan for any 32-bit value in 0x221c0000–0x221cffff: 0 matches
```

The address XBL reads to gate NSP **does not appear in FeatureEnabler at all.** FeatureEnabler is not the writer or provisioner of that value.

---

## Disassembly Confirms No QFPROM Writes

The 61 KB code segment was disassembled (`aarch64-linux-gnu-objdump -D -b binary -m aarch64`). Three independent checks:

1. **No QFPROM address is ever constructed.** No `movz/movk` pair builds any `0x221c....` value. No `adrp` targets any `0x221c` page.

2. **No hardcoded MMIO of any external block.** The `adrp` page-base histogram is entirely internal to the trustlet's own `.rodata`/`.data` pages (`0xb000, 0xc000, 0xd000, 0xe000, 0x11000, 0x12000`). Zero references to MMIO-class pages for MDSS, QFPROM, security controller, or video subsystem. The `HWIOUtils_Write` target (if any) is supplied by QTEE behind `IPCore_OpenModule`, not embedded in code.

3. **The SwFuse does not embed a hardware write address.** Best-supported interpretation: it is an RPMB-license-gated software fuse whose enforcement is mediated through QTEE module dispatch, not through a hardcoded `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB` write.

---

## What FeatureEnabler Cannot Do

| Capability | Answer |
|-----------|--------|
| License NSP/CDSP/NPU/Hexagon compute | **No** — not in its feature catalog |
| Write to QFPROM fuse `0x221c2420` | **No** — zero references to that address space |
| Override the `nsp=0xff` socinfo value | **No** — that value comes from XBL/SMEM before the trustlet runs |
| Provision NSP via RPMB license | **No** — RPMB licenses only cover DisplayCore and VideoCore |
| Read the NSP_DISABLE fuse via `qsee_is_sw_fuse_blown(22)` | **Yes** — but read-only; a blown OTP cannot be unblown |

---

## Why the NPU Is Not a Qualcomm-Licensable Feature

Qualcomm's documented SoftSKU/WES feature pack mechanism covers display and media features only:
- Frame rate limits
- Resolution limits
- Panel quality features (Demura, SPR, AntiAging)

No Qualcomm documentation, product page, or developer resource describes the NPU/Hexagon as a licensable SoftSKU feature. On the Dragonwing Q-8750 (the full-fat AIoT sibling of the CQ8725S/CQ8750S), the **Hexagon V79 NPU at 77 dense TOPS is the headline product feature** — it is not behind a license, it is the reason to buy the chip. NPU capability is a silicon-tier differentiation that lives in fuses, not in RPMB licenses.

> **Note**
> Qualcomm's own support forum identifies the same trustlet as `DisplayFeatureEnablerExecute` on the QCS8550 (a closely related IoT BSP), confirming its display-scoped nature even on industrial/embedded product lines. Reference: [QCS8550 FeatureEnabler thread](https://mysupport.qualcomm.com/supportforums/s/question/0D5dK00000GgsmPSAR/).

---

## Summary

- FeatureEnabler checks `qsee_is_sw_fuse_blown(22)` = `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]` — a hardware OTP fuse read, not a soft fuse write.
- The trustlet's entire feature catalog is `DisplayCore_*` (FPS/res/AntiAging/Demura/SPR) and `VideoCore_*` (resolution limiter). No compute, DSP, NPU, or Hexagon feature exists.
- FeatureEnabler never references `0x221c2420` or any QFPROM fuse aperture — confirmed by string search, byte scan, and disassembly.
- The "SwFuse" mechanism (RPMB-backed, MMIO-enforced) is disjoint from the QFPROM OTP path that gates NSP.

**Verdict from Proof 2: FeatureEnabler closes the "soft unlock" door entirely. It can only read the NSP_DISABLE fuse, never set it — and the NPU is not a Qualcomm-licensable feature on this silicon class.**
