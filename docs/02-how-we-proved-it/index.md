# How We Proved It

This section documents the three independent lines of evidence that converge on a single verdict:

> **The NSP/NPU subsystem on the AYN Odin 3 (CQ8725S-3-AA, SM8750 "sun") is permanently disabled by a factory-blown QFPROM hardware fuse. No software, firmware update, signed feature pack, or HLOS path can reverse it.**

> **These three proofs are the *closing argument*, not the whole case.** They are what finally nailed the verdict — but they sit on top of a **47-phase reverse-engineering campaign** (app-layer probing, firmware RE, five kernel builds, EDL/Firehose forensics, bootloader auth reversing, a brick + recovery, donor teardowns) that first *eliminated every other possibility*. See the **[Full Test Inventory](../03-methods/00-full-test-inventory.md)** for everything tried before it came down to reading the fuse. The verdict isn't a lucky guess that survived — it's the last hypothesis standing after dozens were actively disproven, then confirmed three independent ways.

---

## Why Three Independent Methods?

A single data point can be explained away. A firmware byte could be misconfigured. A kernel node could be wrong. A disassembly trace could be for the wrong SoC family. But when three completely separate investigation approaches — a live on-device read, a trustlet binary analysis, and a cross-reference against kernel source and manufacturer patents — all arrive at the same conclusion, the probability of coincidence collapses.

Each method was designed to be independent: they access different layers of the stack (hardware abstraction → kernel → secure world → microarchitecture documentation), use different tooling, and could in principle disagree. They do not disagree.

---

## The Three-Proof Triangulation

| # | Method | What It Examines | Key Finding | Result |
|---|--------|-----------------|-------------|--------|
| **Proof 1** | Live on-device socinfo read (read-only root, `uid=0`) | `/sys/devices/soc0` partial-goods table, `/proc/iomem`, `/sys/kernel/debug/qcom_socinfo` | `nsp=0xff` in the same hardware binning table that shows `modem=0xff` (verified real bin-out); CDSP firmware image slots empty; raw fuse address `0x221c2420` is outside the HLOS-mapped QFPROM window | NSP subsystem is flagged absent by XBL-written boot data |
| **Proof 2** | FeatureEnabler trustlet static disassembly | SM8750 `featenabler.mbn` binary (Realme GT7 Pro / SM8750 donor; same chipset BSP as Odin 3) | Fuse-check call `qsee_is_sw_fuse_blown(22)` = `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]`; trustlet feature catalog is display + video only; zero references to `0x221c2420` or any QFPROM aperture | FeatureEnabler cannot enable NSP; the NSP gate is a hardware-OTP query, not an RPMB soft fuse |
| **Proof 3** | Kernel source analysis + Qualcomm patent US 12,061,855 | CLO/downstream `socinfo.c`, mainline `socinfo.c`, `SOCINFO_PART_NSP` interface, OTP fuse harvesting patent | `subset_parts` = renamed `defective_parts` array; filled by XBL from QFPROM at boot; indexed by `socinfo_get_subpart_info(SOCINFO_PART_NSP)`; `0xff` = binned-out sentinel; patent confirms OTP fuse blown at manufacturing test | Mechanism is factory-permanent QFPROM OTP; kernel read-path is read-only by design |

**Combined confidence:** P(permanent hardware-disabled via factory QFPROM partial-goods bin-out) ≈ **0.94**.

The residual ~6% covers a single unverified link: that this device's specific CLO kernel build's `nsp` show-function indexes the subset-parts array rather than an alternative `feature_word` path — a plausible path that no evidence contradicts, but which was not independently confirmed against the exact kernel binary.

---

## Reading the Evidence

Each proof page references raw evidence files collected during the 2026-06-30 investigation session. All reads were performed with **no writes** — root access was obtained via PServer Binder (`uid=0 context=u:r:pservice:s0`), and `boot_completed=1` remained true throughout. No device state was modified.

Raw evidence is kept verbatim under `evidence/` in this repository:

- `evidence/fuse-reads/` — on-device socinfo and QFPROM window dumps
- `evidence/feature-enabler/` — trustlet decode findings

> **Note**
> The QFPROM fuse address `0x221c2420` (the raw bit that XBL reads to set the NSP availability byte) **cannot be read from Android**. The HLOS QFPROM nvmem window only covers `0x221c8000–0x221c8fff`, and `/dev/mem` is compiled out (`# CONFIG_DEVMEM is not set`). What we read on-device is the *decoded result* that XBL already published to SMEM — which is authoritative, because XBL is the only entity with access to the full fuse aperture.

---

## Detailed Proof Pages

- [Proof 1 — Live On-Device Socinfo Read](proof-1-socinfo.md)
- [Proof 2 — FeatureEnabler Trustlet Disassembly](proof-2-featenabler.md)
- [Proof 3 — Kernel Source and Qualcomm Patent](proof-3-kernel-patent.md)
