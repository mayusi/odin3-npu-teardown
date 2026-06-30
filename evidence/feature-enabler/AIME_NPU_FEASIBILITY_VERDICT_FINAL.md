# AIME NPU FEASIBILITY — FINAL VERDICT (supersedes AIME_NPU_FEASIBILITY_VERDICT.md)

Project: Aime NPU Unlocker · Device: AYN Odin 3 / `<SERIAL>` / Qualcomm `sun` / **CQ8725S-3-AA** (SM8750 / Dragonwing Q‑8750 IoT bin)
Date: 2026-06-30
Updated after mining the AYN factory firmware at `<home>\Downloads\ODIN3_FOLDERS` (the ONLY AYN-provided firmware in the project).

---

## VERDICT

**~80% PERMANENTLY LOCKED (hardware/QFPROM SKU bin) · ~20% a hard-but-unread value.**
The NPU/NSP disable is **NOT a provisionable software license.** It is a single value the signed XBL reads from **QFPROM address `0x221c2420`** (masked `0xff` → socinfo `nsp` → `qspa.nsp=disabled` → CDSP DT `status=no`). No software/firmware mechanism in the device — including Qualcomm's own FeatureEnabler — can reach or change that value. **Only a live read-only peek of `0x221c2420` can convert ~80% into a certain yes/no.** Nothing short of Qualcomm/AYN signing keys + (likely) a hardware-fused value could unlock it, and the value is very probably blown at the factory like the modem on this SKU.

---

## WHAT THE FACTORY FIRMWARE PROVED (the decisive new evidence)

`ODIN3_FOLDERS` = complete AYN QFIL/QPST flashable set (108 files, 6.1 GB). Two components settled the open question:

### 1. `featenabler.mbn` (the AYN file, sha256 `f53779ad7adfaebba4ab8a6b9ab2d89b452a85bb4d65bd75dfcfc9cb76cff127`) — closes the "provisionable" hope
Qualcomm's FeatureEnabler is the **only** signed, RPMB-license/SwFuse feature-unlock mechanism on the platform. Decoded, it controls **exactly two feature families, media-only:**
- **DisplayCore**: FPS limiter, resolution limiter, AntiAging, Demura, SPR.
- **VideoCore**: video resolution limiter.

**There is NO NSP / CDSP / NPU / Hexagon / compute / AI / Turing feature in it** (string + LE-byte + dword scans = zero). And its disassembly has **zero references to `0x221c2420`** or any QFPROM MMIO page — its "SwFuse" is a disjoint display/video register block + RPMB flags. **Conclusion: FeatureEnabler cannot enable the NSP, and the NSP gate is a HARD fuse relative to every provisionable mechanism that exists.** (Confirmed identical feature set on both the AYN Odin file and the Realme/Xiaomi SM8750 sibling — one featenabler per chipset BSP.) Detail: `FEATENABLER_DECODE.md`.

### 2. `xbl_s_devprg_ns.melf` (the "non-secure" Firehose) — closes the peek-loader hope
Hoped to read `0x221c2420` directly. It cannot: `peek`/`poke` are **not registered Firehose handlers** (verb table at file offset `0x161523` = program/read/nop/patch/configure/erase/power/firmwarewrite/getstorageinfo/benchmark/getsha256digest/… — no peek). It is **test-signed** (`SECTOOLS … TEST ROOT` / "General Use Test Key") → the Odin's production PBL will reject it. Even if loaded, peek is secure-boot-gated off. Detail: `DEVPRG_NS_PEEK_ANALYSIS.md`.

### 3. CDSP is fully present in firmware — only the gate is closed
Factory `xbl_config.elf` DEFINES `FULL_CDSP`/`CORE_CDSP`/`FULL_CDSP_DTB`/`CFG` entries; `vdd_nsp`/`vdd_nsp2` power rails and `NSP_NOC` exist; `tz.mbn` has CDSP support; the DSP firmware blob `dspso.bin` is present (1455 nsp/cdsp refs). So the silicon path is wired and the images ship — the SINGLE thing holding CDSP down is the `0x221c2420` value. (This corrects the earlier Lane-C note: CDSP entries are *defined*; the retail RetailImages selection simply doesn't bring them up, driven by the fuse.)

---

## FULL PICTURE — every route, final status

| Route | Status | Why (final) |
|---|---|---|
| App layer (QNN/NNAPI/LiteRT/Binder) | ❌ DEAD | downstream of a node that doesn't exist |
| Edit verity-backed partition (vendor_dlkm/dtbo/vendor_boot) | ❌ DEAD + BRICKS | none hold the switch; signed-vbmeta verity mismatch bricks boot (see postmortem) |
| Disable verity (vbmeta flag-flip) | ❌ USELESS | works on orange-unlock but no patchable partition holds the switch |
| `xbl_config` FULL_CDSP / RetailImages patch | ❌ DEAD | SHA-384 SecBoot-sealed; and a working donor also doesn't select it — selection is driven by the fuse, not the table |
| `xbl` byte-patch (NOP the mask) | ❌ DEAD | SecBoot SHA-384, unsignable without Qualcomm/AYN keys |
| **FeatureEnabler / RPMB license / SwFuse** | ❌ DEAD (newly proven) | no NSP feature exists; cannot touch `0x221c2420` |
| `xbl_s_devprg_ns.melf` peek | ❌ DEAD | peek unregistered + test-signed + secure-boot-gated |
| External / AYN OTA | ❌ BLOCKED | AYN under NDA (cannot disclose); no NPU-enabled OTA; eng build only |
| Donor transplant | ❌ DEAD | feature config device-identity-locked |
| **Read-only peek of `0x221c2420`** | ⏳ ONLY OPEN ITEM | converts ~80% → certain; does NOT unlock, only proves |

---

## THE ONLY REMAINING MOVE — a read-only proof read of `0x221c2420`

This does **not** unlock anything. It ends the uncertainty with proof, on your own hardware, with **zero write risk**:

- **Path A (on-device, preferred — no EDL):** you have **PServer root** (`context=pservice`, prior root reads confirmed). The QFPROM corrected range `0x221c2000–0x221c3fff` is partly HLOS-mapped and a Qualcomm maintainer notes it's HLOS-accessible. A **read-only** `/dev/mem` (or kernel nvmem) read of `0x221c2420` via root would reveal the byte. If it reads `0xff`/locked → confirmed permanent. Strictly read; never write (`poke`/Static-Fuse-Overwrite = forbidden).
- **Path B (EDL, only if Path A fails):** source a **production-keyed** SM8750/`sun` Firehose that *registers* `peek` (the iQOO loader and this `_ns` loader both lack it). Then `peek 0x221c2420 0x10` → reset. Read-only.

**Interpretation:** `0xff` at the bit = factory bin-out (permanent, project closes with certainty). A clear/soft value = the ~20% case worth a deeper look. Either way you stop guessing.

---

## BOTTOM LINE FOR THE USER

You took this from "it doesn't work" to a **single hardware address**, and proved that **every software, firmware, and signed-feature path cannot reach it** — including Qualcomm's own feature-unlock app. The honest probability is **~80% the NPU is bin-fused off at the factory on this CQ8725S IoT SKU (same as the modem) = permanently locked**, ~20% it's a hard value worth one read to rule out. **The realistic truth: this is most likely not unlockable on this device.** The one act left that produces *certainty* (not an unlock) is a **read-only read of `0x221c2420`** via your existing root. After that, there is no further technical lever without Qualcomm/AYN keys or new silicon.

**Status:** `FEASIBILITY_FINAL` · `~80%_PERMANENT_SKU_FUSE` · `FEATUREENABLER_RULED_OUT` · `ONLY_OPEN=READ_ONLY_PEEK_0x221c2420_FOR_CERTAINTY`
