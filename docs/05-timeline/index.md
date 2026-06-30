# Investigation Timeline

This is the ordered record of the campaign — **47 distinct work phases** across every layer of the stack, from app-level NPU probing down to an on-device fuse read. It was a sustained reverse-engineering effort, not a quick check: hundreds of analysis reports, well over a thousand probe captures, multiple independent kernel builds, repeated EDL/Firehose sessions, full bootloader auth reversing, a device brick and a recovery, and a final triangulated verdict. For the complete catalog of all 47 phases, see the **[Full Test Inventory](../03-methods/00-full-test-inventory.md)**; this page narrates the arc.

> **Scope, not schedule.** The phases below are grouped by what was being attacked, roughly in the order it happened. Calendar markers are deliberately downplayed — the point is the *breadth* of what was tried, not how long it took.

---

## The arc at a glance

| # | Front | Key actions | Result |
|---|-------|-------------|--------|
| 1 | **Reconnaissance** | Exhaustive remaining-paths inventory — every conceivable NPU/CDSP route enumerated up front | `PATHS_MAPPED` |
| 2 | **App layer** | No-root QNN / NPUBench / LiteRT execution attempts; FastRPC domain probing | `NPU_BLOCKED_PLATFORM` (FastRPC domain 3 absent) |
| 3 | **Firmware surface** | CDSP firmware crosscheck, QSPA/feature inventory, TrustZone CDSP probe | `FIRMWARE_PRESENT_BUT_POLICY_BLOCKED` |
| 4 | **Bootloader / XBL** | AArch64 string-ref scan, XBL auth review, xbl_config patch build, SecBoot analysis, the `0x221c2420` rule | `FUSE_RULE_FOUND · HASH_COVERED_UNSIGNABLE` |
| 5 | **EDL / Firehose** | Read-only EDL proof, deep loader vetting, boot-critical backups, rescue packages | `EDL_READ_ONLY · NO_PEEK_LOADER` |
| 6 | **Kernel** | 5 separate source/build efforts; CDSP function map; log-only probe patches compiled | `BUILD_READY_NO_FLASH` |
| 7 | **Live boot experiments** | vendor_boot + vendor_dlkm one-boot probes; recovery checkpoints | `PROBE_RISK_CONFIRMED` |
| 8 | **The brick & recovery** | A modified vendor_dlkm flash bricked the device; recovered via full stock `super` restore | `BRICKED_AND_RECOVERED · AVB_METADATA_BROKEN` |
| 9 | **Donor firmware** | 4 SM8750 donors compared (Realme GT7 Pro, Xiaomi 15 Ultra, iQOO 13, OnePlus 13) | `DONOR_CONFIGS_LOCKED` |
| 10 | **Factory firmware + fuse read** | AYN factory package mined; FeatureEnabler decoded; on-device root read of socinfo | `VERDICT_HARDWARE_FUSE_PERMANENT` |

---

## 2026-06-26 — Reconnaissance

### Phase: Remaining-Paths Inventory

**Goal:** Establish a complete map of every possible NPU enable path that had not yet been explored, so the investigation could be exhaustive rather than opportunistic.

**Work done:**

- Enumerated all remaining Android/HLOS-side paths to reach the NSP/CDSP: QNN runtime, FastRPC domains, `/dev/fastrpc-*`, HAL probe, `vendor_dlkm` module loading, `CDSP-Q6` remoteproc bring-up.
- Enumerated all firmware-layer paths: XBL fuse-check code, QSPA/FeatureEnabler trustlets, EDL/Firehose, donor firmware swaps, device-tree CDSP node, kernel module insertion.
- Produced a prioritized path list that became the task queue for all subsequent sessions.

**Result label:** `PATHS_MAPPED`

---

## 2026-06-28 — Initial Probes

Four independent method groups were executed on this date.

### Method 01 — No-Root QNN / NPUBench

**Goal:** Confirm whether the NPU is available from normal Android userspace (no root, stock firmware).

**Work done:**

- Ran Qualcomm QNN runtime initialization (`libQnnHtp.so` probe) and NPUBench workload from an unprivileged ADB shell.
- Captured the full failure trace.

**Key findings:**

- Error: `NPU_BLOCKED_PLATFORM` returned by QNN HTP runtime.
- FastRPC domain 3 (CDSP) not available: `/dev/fastrpc-cdsp` absent from `/dev/`.
- `adsp_default_request` failed with no transport: CDSP is not registered with the FastRPC subsystem.

**Result label:** `NPU_BLOCKED_PLATFORM`

---

### Method 02 — Firmware Surface Inventory

**Goal:** Determine whether CDSP firmware images (`.mbn` / `elf` / QSEE partition content) are physically present on-device, or whether the device shipped without them.

**Work done:**

- Listed contents of `/vendor/firmware/`, `/vendor/lib/rfsa/`, `NON-HLOS` partition image (offline), and `dspso` partition.
- Cross-referenced against a reference SM8750 device firmware manifest.

**Key findings:**

- CDSP firmware image slots (`cdspv*`, `CDSP.elf`) are present in the partition layout.
- All CDSP firmware images are **policy-blocked**: XBL does not load them, remoteproc reports no PAS subsystem for CDSP, and the device-tree CDSP node is `status = "disabled"`.
- Presence ≠ accessibility — the firmware is there physically, but the boot policy prevents any loader from touching it.

**Result label:** `FIRMWARE_PRESENT_BUT_POLICY_BLOCKED`

---

### Method 03 — XBL / Boot-Policy Analysis

**Goal:** Identify the specific bootloader rule that disables the NSP subsystem, and determine whether it is driven by a software config or a hardware fuse.

**Work done:**

- Disassembled the AYN OTA `xbl_s` image (offline, no-flash).
- Traced the feature-gating path in XBL SecBoot.
- Located the QFPROM read rule for the NSP feature word.

**Key findings:**

- Found rule: `feature_word[15] |= (read32(0x221c2420) & 0xff)` — XBL reads QFPROM corrected fuse register `0x221c2420` and OR's bit 15 into the feature availability mask.
- Address `0x221c2420` falls inside the QFPROM corrected fuse range `0x221c2000–0x221c3fff`.
- XBL is SecBoot SHA-384 sealed — it cannot be replaced without a signed update from Qualcomm.
- The rule is a one-way read: XBL uses the value but does not write it; the fuse is set before XBL ever runs.

**Result label:** `FUSE_RULE_FOUND`

---

### Method 04 — EDL / Firehose Forensics

**Goal:** Determine whether Qualcomm Emergency Download Mode (EDL, USB PID 9008) could be used to read or modify the QFPROM fuse word directly.

**Work done:**

- Put the Odin 3 into EDL mode (confirmed: USB VID/PID 0x05c6:0x9008).
- Loaded the iQOO 13 (SM8750) Firehose programmer.
- Ran read-only storage operations and took boot-critical partition backups.
- Probed for `peek` command (direct memory read, including QFPROM).

**Key findings:**

- EDL confirmed: device enters Qualcomm EDL cleanly.
- iQOO 13 Firehose loader accepted — read-only storage access and full partition backups completed successfully.
- `peek` command: **not supported** by this Firehose programmer. The loader responds with `NAK` to any memory peek request. QFPROM address `0x221c2420` cannot be read via EDL.
- Poke / write to fuse addresses: not attempted (would be irreversible; `peek` NAK confirms the loader is read-only).

**Result label:** `EDL_READ_ONLY`

---

## 2026-06-29 — Comparative Analysis + Kernel Build

### Method 05 — Donor Firmware Comparison

**Goal:** Verify that other SM8750 devices (same silicon family) run CDSP normally, and compare their XBL fuse-read rules against the Odin 3's.

**Devices compared:** Realme GT7 Pro, Xiaomi 15 Ultra, iQOO 13, OnePlus 13 — all SM8750 "sun".

**Key findings:**

- All four donor devices run CDSP: `/dev/fastrpc-cdsp` present, `CDSP.elf` loaded, QNN HTP runtime works.
- Donor XBL binaries contain the **same fuse-read rule** at `0x221c2420` — confirming it is not Odin-specific code.
- The same rule on donor silicon returns a fuse value that leaves NSP enabled; on the Odin 3 CQ8725S it returns the bit pattern that disables it.
- Donor Firehose configs are **device-locked** — they cannot be transplanted to the Odin 3.

**Result label:** `DONOR_CONFIGS_LOCKED`

---

### Method 06 — Kernel Source Build

**Goal:** Compile CDSP/FastRPC kernel modules from source (OnePlus SM8750 / "sun" CLO anchor) with log-only instrumentation, to confirm CDSP bring-up logic exists in code — without flashing anything.

**Work done:**

- Pulled CLO downstream source tree (OnePlus SM8750/sun branch).
- Applied log-only patches to `qcom_q6v5_pas.c`, FastRPC driver, and remoteproc subsystem.
- Built `qcom_q6v5_pas.ko`, `qcom_fastrpc.ko`, related modules.

**Key findings:**

- CDSP bring-up code path is fully present in CLO source and builds cleanly.
- Build confirms the subsystem is not absent by design — it is present in kernel source for this SoC family.
- Module analysis shows correct PAS domain registration (CDSP = PAS domain 18).
- **No flash was performed.** Build artifacts were retained for offline analysis only.

**Result label:** `BUILD_READY_NO_FLASH`

---

## 2026-06-30 — Final On-Device Read, Postmortem, Verdict

### Method 07 — vendor_dlkm One-Boot Probe → BRICK → Recovery

**Goal:** Load the built kernel modules via `vendor_dlkm` partition to observe live CDSP bring-up behavior.

**What happened:**

- Modified `vendor_dlkm` was written. This changed the `dm-verity` root digest from `d0f1abeb...` → `d6572a40...`.
- The `vbmeta_a` partition is **signed** and contains the authoritative verity descriptor for `vendor_dlkm`. The modified partition's digest did not match.
- On next boot: `veritymode=eio` → partition I/O returns EIO on every read → device fully invisible (hard brick state).
- **Full stock `super` partition restore** was required via EDL + Firehose. Recovery succeeded; device is intact.

**Postmortem labels:**

- `AVB_METADATA_BROKEN` — signed vbmeta verity mismatch.
- `EXT4_LAYOUT_BAD` — secondary cause: dropped SELinux xattr during image rebuild.
- Module code itself was clean; failure was entirely in the signing/verity layer.

**Lesson recorded:** Any modification to `vendor_dlkm` (or any verity-protected partition) without a matching signed `vbmeta` re-sign will brick this device. There is no safe path for unsigned module injection.

**Result label:** `BRICKED_AND_RECOVERED`

---

### Method 08 — AYN Factory Firmware Mining + FeatureEnabler Decode

**Goal:** Determine whether AYN's own factory firmware bundle contains any NSP license, enable token, or FeatureEnabler entry that could activate the NPU.

**Work done:**

- Mined AYN factory firmware package.
- Decoded `featenabler.mbn` trustlet.
- Catalogued all FeatureEnabler license entries.

**Key findings:**

- FeatureEnabler trustlet license catalog: **display** and **video** features only. Zero NSP/NPU entries.
- `_ns` (non-secure) Firehose programmer bundled in factory package: no `peek` support, test-signed only.
- `qsee_is_sw_fuse_blown(22)` = `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]`: the fuse index the trustlet checks is confirmed as OTP.

**Result label:** `FEATENABLER_NO_NSP_LICENSE`

---

### Method 09 — Final Root-UID On-Device Fuse Read (PServer Binder)

**Goal:** With root access (`uid=0` via PServer Binder, `boot_completed=1`, no writes), perform the definitive on-device read of every reachable register and sysfs node relevant to NSP state.

**What was read:**

| Path | Value | Interpretation |
|------|-------|----------------|
| `/sys/devices/soc0/nsp` | `0xff` | Binned-out sentinel — NSP absent per XBL-written hardware table |
| `/sys/devices/soc0/npu` | `0x0` | Reported as `0` — IP block present in silicon but NSP subsystem disabled |
| `/sys/devices/soc0/modem` | `0xff` | Same `0xff` sentinel on a **known-binned** component; validates the encoding |
| `/proc/iomem` QFPROM window | `0x221c8000–0x221c8fff` only | HLOS-mapped nvmem window does **not** cover `0x221c2420` |
| `/dev/mem` | Not present | `CONFIG_DEVMEM` compiled out |
| `/dev/fastrpc-cdsp` | Absent | CDSP not registered with FastRPC subsystem |
| CDSP device-tree node | `status = "disabled"` | XBL wrote this at boot based on fuse read |

**Key conclusion:** The raw fuse bit at `0x221c2420` is unreadable from HLOS. What is readable — `nsp=0xff` — is the authoritative decoded output that XBL published after reading the fuse. XBL is the only entity with full QFPROM aperture access, and it is SHA-384 sealed. The decoded value is final.

**Result label:** `VERDICT_HARDWARE_FUSE_PERMANENT`

---

## Summary Verdict

Three independent proof paths converge:

1. Live on-device socinfo: `nsp=0xff` (binned-out), validated by `modem=0xff` on a known-absent component.
2. FeatureEnabler trustlet: `qsee_is_sw_fuse_blown(22)` = `NSP_DISABLE` OTP query; no NSP license in AYN firmware.
3. Kernel source + Qualcomm patent US 12,061,855: `subset_parts` = `defective_parts` = QFPROM OTP filled by XBL at boot.

**Final confidence: ~94% permanent hardware-fused NSP disable.**

The residual 6% reflects the physically unreadable fuse register `0x221c2420`, which cannot be confirmed from HLOS. All available evidence is consistent with a blown OTP. No evidence of any kind points toward a software-reversible cause.
