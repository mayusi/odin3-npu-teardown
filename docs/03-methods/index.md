# Methods — Everything We Tried

The investigation ran across **47 distinct work phases** — the complete catalog is in the **[Full Test Inventory](00-full-test-inventory.md)** (every phase, with file counts). The pages below are the **nine deep-dive write-ups** of the most significant fronts: each describes the goal, the tooling, the exact steps, and the result — including what it could and could not rule out. They are the spine of the campaign, but they are *highlights of a much larger body of work*, not the whole of it.

> **Start with the [Full Test Inventory](00-full-test-inventory.md)** if you want the full scope — app-layer probing, firmware RE, five separate kernel builds, multiple EDL/Firehose sessions, bootloader auth reversing, donor teardowns, a brick + recovery, and the final fuse read. The nine pages here go deep on the load-bearing ones.

---

## Index

| # | Method | Date | One-Line Summary | Result |
|---|--------|------|-----------------|--------|
| [01](01-no-root-qnn.md) | No-Root QNN / NPUBench | 2026-06-28 | QNN HTP runtime probe from unprivileged ADB shell — CDSP not present, FastRPC domain 3 absent | `NPU_BLOCKED_PLATFORM` |
| [02](02-firmware-inventory.md) | Firmware Surface Inventory | 2026-06-28 | Catalogued all CDSP firmware image slots on-device and in partition dumps — images present but policy-blocked at the boot layer | `FIRMWARE_PRESENT_BUT_POLICY_BLOCKED` |
| [03](03-xbl-boot-policy.md) | XBL / Boot-Policy Analysis | 2026-06-28 | Disassembled `xbl_s` offline — found QFPROM fuse read rule at `0x221c2420` gating NSP, XBL is SHA-384 SecBoot sealed | `FUSE_RULE_FOUND` |
| [04](04-edl-firehose.md) | EDL / Firehose Forensics | 2026-06-28 | Device entered EDL (USB 0x05c6:0x9008); iQOO 13 Firehose loaded — read-only storage and backups only; `peek` not supported | `EDL_READ_ONLY` |
| [05](05-donor-firmware.md) | Donor Firmware Comparison | 2026-06-29 | Cross-referenced XBL fuse rules and CDSP availability across four SM8750 phones (Realme GT7 Pro, Xiaomi 15U, iQOO 13, OnePlus 13) — same rule, different fuse value; donor configs device-locked | `DONOR_CONFIGS_LOCKED` |
| [06](06-kernel-build.md) | Kernel Source Build | 2026-06-29 | Built CDSP/FastRPC modules from CLO OnePlus/Sun source with log-only patches — confirms bring-up code is present; no flash performed | `BUILD_READY_NO_FLASH` |
| [07](07-vendor-dlkm-probes.md) | vendor_dlkm Module Probe | 2026-06-30 | Attempted to load built modules via `vendor_dlkm` partition — modified image changed dm-verity digest, mismatched signed `vbmeta_a` | `BRICKED_AND_RECOVERED` |
| [08](08-brick-and-recovery.md) | Brick & Full Recovery | 2026-06-30 | Postmortem of the vendor_dlkm brick: `AVB_METADATA_BROKEN` (verity digest mismatch) + `EXT4_LAYOUT_BAD` (dropped SELinux xattr); recovered via full stock `super` restore over EDL | `DEVICE_RESTORED` |
| [09](09-featenabler-decode.md) | FeatureEnabler Decode | 2026-06-30 | Decoded AYN factory `featenabler.mbn` and SM8750 donor trustlet — license catalog is display/video only; `qsee_is_sw_fuse_blown(22)` = `NSP_DISABLE` OTP query; no NSP license path exists | `FEATENABLER_NO_NSP_LICENSE` |

---

## Reading a Method Page

Each method page follows the same structure:

- **Goal** — what question this method was designed to answer.
- **Prerequisites** — tooling, root status, which firmware was used.
- **Steps** — the exact commands or analysis steps, with representative output.
- **Findings** — what the method revealed, including null results.
- **Limitations** — what the method cannot rule out, and why.
- **Evidence references** — links to raw artifacts in `evidence/`.
- **Result label** — a single identifier summarizing the outcome.

---

## Why Nine Methods?

Each method attacks a different layer of the stack:

```
Layer 7 — Android userspace     → Method 01 (QNN runtime probe)
Layer 6 — Vendor firmware/HAL  → Method 02 (firmware inventory)
                                 Method 09 (FeatureEnabler trustlet)
Layer 5 — Kernel / modules      → Method 06 (kernel source build)
                                 Method 07 (vendor_dlkm probe)
Layer 4 — Bootloader (XBL)     → Method 03 (XBL disassembly)
Layer 3 — Secure world (QSEE)  → Method 09 (trustlet disassembly)
Layer 2 — EDL / Firehose       → Method 04 (EDL forensics)
Layer 1 — Comparative silicon  → Method 05 (donor firmware)
Layer 0 — Hardware fuse (OTP)  → Methods 01–09 all converge here
```

A result at any single layer could have a layer-local explanation (misconfiguration, wrong firmware, missing HAL). The convergence across all nine methods — each independent, each accessing a different abstraction layer — is what drives the ~94% confidence in the hardware-fuse conclusion.

---

## The Method That Taught Us the Most (and Hurt the Most)

Method 07 (vendor_dlkm probe) was the only write operation attempted. It ended in a brick and a full EDL recovery. The lesson it provided was unambiguous: the signed `vbmeta` AVB chain is enforced unconditionally on this device, and there is no safe path for unsigned partition modifications. This closes the last plausible "software path" — not because the software path was impossible to attempt, but because attempting it confirms there is no exit that doesn't require Qualcomm's signing keys.

See [`evidence/vendor-dlkm-postmortem/`](../../evidence/vendor-dlkm-postmortem/) for the full postmortem report.
