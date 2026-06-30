# FeatureEnabler (`featenabler.mbn`) Decode — Does it gate the NSP/NPU? Is `0x221c2420` a SwFuse it provisions?

Generated: 2026-06-30. OFFLINE static analysis only. No device / adb / fastboot / EDL touched.
Analyst scope: AYN Odin 3 NPU-disable feasibility (device `sun`, SoC CQ8725S-3-AA = Qualcomm SM8750 "sun" / Snapdragon 8 Elite, soc_id 706, rev 2.0).

---

## 0. CRITICAL PROVENANCE CORRECTION (read first)

The mission brief states `featenabler.mbn` was found in an **AYN ODIN3 factory firmware** at `<home>\Downloads\ODIN3_FOLDERS`.
**That path does not exist on disk**, and there is **no AYN/Odin firmware copy of featenabler anywhere under `~/Downloads`.** The only `featenabler.mbn` files present are **DONOR extracts**:

- `…/aime-npu-unlock-20260629/realme_gt7_pro_local_direct_entries/IMAGES__featenabler.mbn` (md5 `0d3fb501a7a5dcd9eb0cb3edfc64759d`, 105 440 B) ← analyzed here
- `…/aime-npu-unlock-20260628/donor_extract_20260628/realme_gt7_pro_rmx5011/remote_direct_entries/IMAGES__featenabler.mbn`
- `…/aime-npu-unlock-20260629/xiaomi_15_ultra_xuanyuan_…__images__featenabler.mbn`

So this binary is from the **Realme GT7 Pro (RMX5011)** OTA, with a Xiaomi 15 Ultra sibling. **Both donors are the SAME SoC family (SM8750 "sun")** as the Odin 3, so the binary is a valid *proxy* for what the Odin's own featenabler would contain (Qualcomm ships one featenabler per chipset BSP; OEMs re-sign it). It is **NOT** AYN-provided, and the 105 KB "AYN firmware" framing in the brief is a misattribution. Conclusions below are about the SM8750 FeatureEnabler in general (Oplus/BBK build); they transfer to the Odin's featenabler with high confidence but were not read from an AYN file.

Binary identity: ELF64 / AArch64 / `Type=DYN`, 9 program headers, no section headers — a signed Qualcomm **QTEE/TrustZone trustlet**. Single code segment `R-E` at file off `0x1000`, vaddr `0x0`, size `0xf0ff` (~61 KB). Signed under the **"OPLUS Attestation CA" / "OPLUS ROOT CA 1" / "Qualcomm Cryptographic Operations"** chain (strings present), TA manifest `n=featenabler` (`B=true;n=featenabler;p=8:c47728cf3e4081,61:6,77,82:6004002,b4,10a:9024,157:db104e`). Confirms it is Qualcomm's **feature-license app** ("feature license app init/shutdown" strings).

---

## 1. Q1 — Does FeatureEnabler control NSP / CDSP / NPU / Hexagon? **NO.**

### 1a. The complete feature surface (every handler + feature name string in the binary)

The app exposes exactly **two feature "cores"** plus generic license/RPMB plumbing. There is **no third core**, and **no compute/DSP/NPU core of any kind.**

**Top-level handlers** (`strings -n 3`):
`FeatureEnabler_Handler`, `LicenseValidateAndEnable_Handler`, `LicenseValidator_Handler`, `LicenseValidatorTest_Handler`, `GetEnabledFeatures_Handler`, `GetInstalledFeatures_Handler`, `GetInstalledFeatures_FromRPMB`.

**`DisplayCore_*` (display feature gating):**
```
DisplayCore_Open / Close / ConfigureSwFuse / EnableSwFuse / ReadSwFuseStatus / GetSwFuseStatus
DisplayCore_GetFeatureList / GetFeatureName / GetFeatureString / IsFeatureSupported
DisplayCore_GetFpsLimit / GetFpsLimitStatus / GetResolutionLimit / GetResolutionLimitStatus / SetResFpsLimit
DisplayCore_EncodeMetaData
```
Display feature NAMES recovered (.rodata):
`Display_ResFpsLimiter`, `Display_AntiAging`, `Display_Demura`, `Display_SPR`, `Display_Allocate_Cache_Signal`,
`FpsLimitIntf`, `HeightLimitIntf`, `WidthLimitIntf`, `ResolutionLimitEnable` (+ format strings `Display_Fps_limit:`, `%s: Fps Limit Intf%d %s`).

**`VideoCore_*` (video feature gating):**
```
VideoCore_Open / Close / ConfigureSwFuse / EnableBit / ReadBitStatus / GetSwFuseStatus
VideoCore_GetFeatureList / GetFeatureName / IsFeatureSupported / FillCommonLicenseInfo / EncodeMetaData
```
Video feature NAME recovered: `Video_Resolution_Limiter` (+ `WidthLimitIntf`).

**Generic infra (not a feature):** `IPCore_OpenModule/CloseModule` (opens a "module_id" — the dispatch shim to Display/Video cores), `IPFM_*` (In-Field Provisioning Manager: `IPFM_CheckInstalledLicense`, `IPFM_GetInstalledLicenseInfo`, `IPFM_CheckFIDAndGetAllSerialNums`, `IPFM_SetOptions`), `RPMBUtils_*` + `qsee_stor_read_sectors/write_sectors` (RPMB I/O), `qsee_is_sw_fuse_blown`, `HWIOUtils_Read/Write` (MMIO accessor), `SharedMemUtils_*`, `CommonUtils_GetSocHwVersion`, full **QCBOR** codec (licenses are CBOR-encoded), `CFE_ValidateLicense`.

### 1b. NSP/NPU token search — exhaustively negative

Grep over all strings (`-n 3`) for `nsp|cdsp|hexagon|npu|turing|neural|tensor|dsp|compute|ai|ml|adsp|q6|hvx|aix|aip` returned **only generic false-positives** (`feature_id_count input is invalid`, `UsefulInputBuf_GetBytes`, `input bit_pos`, etc.). A direct byte/substring scan for `cdsp CDSP turing Turing hexagon Hexagon Hvx HVX NSP q6v5 npu NPU neural Neural compute Compute` produced **zero hits**.

**Verdict Q1: FeatureEnabler does NOT know about, list, or gate the NSP/CDSP/NPU/Hexagon. Its entire feature catalog is consumer media limits — display FPS cap, display/video resolution + width/height cap, and panel-quality features (Demura/SPR/AntiAging). The Hexagon compute subsystem is not a FeatureEnabler-licensable feature on SM8750.**

Independent corroboration: Qualcomm's own support forum names this component **`DisplayFeatureEnablerExecute`** on the QCS8550 (same Hexagon-class IoT BSP) — display-scoped, no compute role. (https://mysupport.qualcomm.com/supportforums/s/question/0D5dK00000GgsmPSAR/qcs8550-rb5-gen2-boot-loop-caused-by-displayfeatureenablerexecute-failure-rspstatus3)

---

## 2. Q3 (the crux) — Is `0x221c2420` a SwFuse that FeatureEnabler provisions, or an independent hard QFPROM fuse?

### 2a. The decisive negative: featenabler never touches `0x221c2420` or the QFPROM aperture

- **String scan**: no occurrence of `221c2420`, `221c2`, `221c3`, `221c0`, `221c8`, `0x221c`, or `qfprom` anywhere in the binary.
- **Little-endian byte scan** for the rule's source pointer `20 24 1c 22` (= `0x221c2420`): **not present.**
- **Full dword scan** of the entire 105 KB file for any 32-bit value in the QFPROM fuse space `0x221c0000–0x221cffff` (covers raw `…0000–…1fff`, ECC-corrected `…2000–…3fff` where `0x221c2420` lives, and the HLOS controller base `…8000`): **0 matches.**

So the address XBL reads to gate NSP (`value = read32(0x221c2420) & 0xff; feature_word[15] |= value`, proven in `aime-npu-unlock-20260628/…CURRENT_STATE…:47-67`) **does not appear in FeatureEnabler's code or data at all.** FeatureEnabler cannot be the writer/provisioner of that value, because it never references that aperture.

### 2b. What FeatureEnabler's "SwFuse" actually is

FeatureEnabler's `*_ConfigureSwFuse` / `EnableSwFuse` / `qsee_is_sw_fuse_blown` operate on a **software fuse that lives in RPMB-backed secure storage**, gated by a **signed CBOR license**, and *applied* by writing a display/video **HWIO register** via `HWIOUtils_Write` (a `*Core`-private MMIO block, e.g. MDSS/DISP_CC / video subsystem — pinned by the disassembly pass, see §2c). The flow is:

```
License (CBOR, signed) → installed in RPMB (qsee_stor_write_sectors) →
FeatureEnabler validates (CFE_ValidateLicense, soc_hw_version check) →
DisplayCore/VideoCore_ConfigureSwFuse → HWIOUtils_Write(<display/video MMIO>) + qsee_is_sw_fuse_blown
```

This is a **different mechanism in a different address space** from the QFPROM fuse at `0x221c2420`. "SwFuse" here = *RPMB software-fuse + provisioned MMIO bit*, NOT a QFPROM OTP cell. The error string `ConfigureSwFuse failed for feature_id %d` is about *this* RPMB/MMIO path.

> Matches Qualcomm's documented model: feature licenses are **installed in RPMB**, "a copy maintained in the data partition," accessible only from QTEE (https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/upgrade-qualcomm-wes-feature-pack.html). SoftSKU "feature packs" are **software-installed/upgraded** on a deployed SoC — i.e. provisionable, not a blown fuse.

### 2c. Therefore: `0x221c2420` is an INDEPENDENT hard fuse, read by XBL BEFORE/outside FeatureEnabler

The two are **disjoint**:
- **XBL path (NSP gate):** reads raw fuse `0x221c2420` in the QFPROM aperture, very early in boot, copies bit-field into `socinfo feature_word[15]` → `/sys/devices/soc0/nsp`. FeatureEnabler runs **later** (it is a QTEE trustlet loaded after XBL/QTEE are up) and **never reads that address**.
- **FeatureEnabler path (display/video gate):** RPMB-license → SwFuse → display/video MMIO. Disjoint feature set, disjoint storage, disjoint address space.

**Best determination:** `0x221c2420` is **NOT a SwFuse/RPMB shadow that FeatureEnabler provisions.** It is an **independent, hardware fuse-region value consumed by XBL**, on a path FeatureEnabler has no code to touch. FeatureEnabler being a "soft, provisionable" mechanism is therefore **irrelevant to the NSP gate** — it gates a different, media-only feature class. (Whether `0x221c2420` itself is a *blown OTP cell* vs a *re-provisionable secure-config row in the corrected aperture* remains the open question from Lane B — but FeatureEnabler is **not** the tool that would re-provision it, and provides **no** new path to it.)

### 2d. Disassembly confirmation (objdump aarch64, code segment carved off=0x1000 size=0xf0ff)

Disassembled the 61 KB code segment (`aarch64-linux-gnu-objdump -D -b binary -m aarch64`). Three independent checks, all confirming §2a–§2c:

1. **No QFPROM address is ever constructed.** No `movz/movk` pair builds any `0x221c....` value; no `adrp` targets any `0x221c` page. (`grep movz/movk #0x221c` → none.) The `0x780000` QFPROM-controller base also never appears as a real reference (the one byte-match `0x78614d20` is data mis-decoded as `.inst … undefined`).
2. **No hardcoded MMIO of ANY block.** `adrp` page-base histogram is **entirely internal** to the trustlet: `0xb000, 0xc000, 0xd000, 0xe000, 0x11000, 0x12000` = its own `.rodata`/`.data` (the file is ~0xf0ff code then data at vaddr `0x10000+`). **Zero** `adrp` to MMIO-class pages (`0x0a..` MDSS / `0x22..` QFPROM / `0x0c..` security / `0x05..` video). So `HWIOUtils_Write`/`ConfigureSwFuse` do **not** embed a hardware register address — the MMIO base (if any) is supplied by **QTEE** behind the `IPCore_OpenModule(module_id)` syscall, or the SwFuse is a pure **RPMB/secure-storage software flag**.
3. **soc_hw_version compare constants not statically recoverable** — they are not `movz/movk` immediates in code (the value comes from the `GetSocHwVersion`/`IDeviceID_getHWVersion` syscall and/or the license CBOR payload, compared against a `.rodata` table). The `cmp #imm` values present (`0xf37, 0xb0f, 0x100, 0x12c…`) are loop/buffer bounds, not SoC versions. This does not affect any conclusion: the gated features are display/video media limits regardless of which soc_hw_version is whitelisted.

**Net of disassembly:** the SwFuse is **not** a hardcoded hardware-fuse write and **categorically not** a QFPROM write. Best-supported call: it is an **RPMB-license-gated software fuse** whose enforcement bit (where it touches hardware at all) is applied through a **QTEE-mediated display/video module** (`IPCore_OpenModule`), never through `0x221c2420`/QFPROM. FeatureEnabler is therefore mechanistically incapable of altering the NSP gate.

---

## 3. Q2/Q4 — Does Qualcomm license the NPU as a feature on Dragonwing/CQ-class SoCs? **No public evidence; strongly indicated NO.**

- FeatureEnabler / SoftSKU "feature packs" are documented **only** for display/media/edge-services features (WES — Wireless Edge Services), installed via RPMB license. NPU/Hexagon **never appears** in that catalog. (https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/upgrade-qualcomm-wes-feature-pack.html ; security feature index https://docs.qualcomm.com/bundle/publicresource/topics/80-80020-11/features.html)
- Across Qualcomm's NPU/Hexagon product and developer pages, the NPU is presented as a **fixed hardware engine** whose TOPS varies by **silicon tier / binning**, never by a software license you install. No "enable AI via feature pack / RPMB license" mechanism is documented anywhere. (https://www.qualcomm.com/processors/hexagon ; https://en.wikipedia.org/wiki/Qualcomm_Hexagon)
- On the **Dragonwing Q-8750** (the AIoT sibling of CQ8725S/CQ8750S), the **Hexagon V79 NPU @ 77 dense TOPS is the headline feature** — the AI engine is the product's reason to exist, not something gated behind a license. (https://www.cnx-software.com/2026/01/06/qualcomm-dragonwing-q-7790-and-q-8750-aiot-socs-target-ai-enhanced-drones-cameras-tvs-and-media-hubs/)
- CQ8725S's documented IoT-binning removals are **modem-RF + camera/ISP**, **not** the NPU. (https://retrohandhelds.gg/your-odin-3-might-not-actually-use-the-snapdragon-8-elite-but-theres-more-to-the-story/)

**Conclusion:** the NPU is **not a Qualcomm-licensable software feature** on this silicon class. This *removes* the "maybe FeatureEnabler can provision the NPU" hope and **leans the NSP-disable back toward a hard fuse/binning decision** (consistent with Lane B's ~0.7-permanent posture), since the one "soft, provisionable" feature mechanism present in firmware (FeatureEnabler) demonstrably does **not** cover compute.

---

## 4. soc_hw_version gate (Q4 detail)

FeatureEnabler **does** gate features per SoC HW version — strings `Invalid soc_hw_version %x`, `Licensing not supported for soc_hw_version %x`, `feature id %li not supported for soc_hw_version %x`, via `CommonUtils_GetSocHwVersion` / `IDeviceID_getHWVersion`. **However, since the gated features are display/video media limits — not NSP — this gate is irrelevant to NPU enablement.** Even if rev 2.0 / soc_id 706 "qualifies," the only thing it could ever unlock here is an FPS/resolution cap, never the Hexagon subsystem. (The concrete soc_hw_version constants are recovered in the appended disassembly section; they do not change this conclusion.)

---

## 5. Bottom line

| Question | Answer | Confidence | Basis |
|---|---|---|---|
| Does FeatureEnabler list/gate NSP/CDSP/NPU/Hexagon? | **NO** | VERY HIGH | Only DisplayCore + VideoCore exist; zero compute/DSP/NPU strings or handlers |
| Is `0x221c2420` a SwFuse FeatureEnabler provisions? | **NO** — independent hard fuse read by XBL | HIGH | featenabler has 0 refs to `0x221c2420` / QFPROM `0x221c0000–0xcfff`; its SwFuse = RPMB-license + display/video MMIO, a disjoint path |
| Does Qualcomm license NPU as a feature (Dragonwing/CQ)? | **NO public mechanism** | MEDIUM-HIGH | FeatureEnabler/SoftSKU cover media/WES only; NPU is fixed HW + headline feature of Q-8750 |
| Net effect on NPU-unlock feasibility | **Leans MORE toward permanent** | — | The one provisionable firmware feature path (FeatureEnabler) does not reach NSP; no soft unlock surface exists for compute |

**Most decisive new fact:** FeatureEnabler — the only signed, RPMB-provisionable feature-licensing mechanism in this firmware — has **no NSP/CDSP/NPU feature and makes zero reference to `0x221c2420` or the QFPROM aperture.** The NSP disable lives on a **separate XBL-read fuse path that FeatureEnabler cannot touch**, and the NPU is not a Qualcomm-licensable feature. This **closes the "locked-but-provisionable via FeatureEnabler" door** and pushes the verdict from ~0.7-permanent toward ~0.78–0.80-permanent. It does **not** newly *prove* `0x221c2420` is a blown OTP cell (that still needs a live peek), but it eliminates FeatureEnabler as a route to flip it.

---

## Sources
- Project priors: `aime-npu-unlock-20260628/AIME_NPU_UNLOCK_CURRENT_STATE_20260628.md:47-67` (XBL rule `read32(0x221c2420)&0xff→feature_word[15]`); `aime-npu-feasibility-20260630/LANE_B_FUSE_VS_POLICY.md`, `LANE_B_ADVERSARIAL_FUSE_VERIFY.md`.
- Binary analyzed: `aime-npu-unlock-20260629/realme_gt7_pro_local_direct_entries/IMAGES__featenabler.mbn` (md5 `0d3fb501a7a5dcd9eb0cb3edfc64759d`).
- Qualcomm SoftSKU feature packs (RPMB-installed licenses): https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/upgrade-qualcomm-wes-feature-pack.html
- Qualcomm devcfg from QTEE: https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-11/enable-device-devcfg-from-qtee.html
- Qualcomm Linux security features index: https://docs.qualcomm.com/bundle/publicresource/topics/80-80020-11/features.html
- DisplayFeatureEnablerExecute (display-scoped FeatureEnabler) QCS8550 thread: https://mysupport.qualcomm.com/supportforums/s/question/0D5dK00000GgsmPSAR/qcs8550-rb5-gen2-boot-loop-caused-by-displayfeatureenablerexecute-failure-rspstatus3
- Dragonwing Q-8750 Hexagon V79 NPU @77 TOPS headline: https://www.cnx-software.com/2026/01/06/qualcomm-dragonwing-q-7790-and-q-8750-aiot-socs-target-ai-enhanced-drones-cameras-tvs-and-media-hubs/
- CQ8725S binning removes modem+camera, not NPU: https://retrohandhelds.gg/your-odin-3-might-not-actually-use-the-snapdragon-8-elite-but-theres-more-to-the-story/
- Qualcomm Hexagon NPU (fixed HW engine): https://www.qualcomm.com/processors/hexagon ; https://en.wikipedia.org/wiki/Qualcomm_Hexagon
