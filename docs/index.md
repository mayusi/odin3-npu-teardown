# Odin 3 / CQ8725S — NPU (NSP) Unlock Investigation

### A complete, evidence-backed teardown of *why* the Hexagon NPU on the AYN Odin 3 cannot be enabled — and everything that was tried to enable it.

**Device:** AYN Odin 3 · **SoC:** Qualcomm `sun` / SM8750-class, model **CQ8725S-3-AA** (Dragonwing / IoT bin)

!!! danger "Verdict: `NSP_HARDWARE_FUSE_DISABLED`"
    The NPU subsystem is **fused off at the factory**, the same way the modem is. It is **not unlockable** by software, firmware, signing, or any on-device path (**~94% confidence**).

---

> **TL;DR.** The Odin 3 ships with `/sys/devices/soc0/nsp = 0xff` (NSP subsystem reported absent) while `npu = 0x0`. This investigation proves — with on-device reads, firmware reverse-engineering, kernel-source attribution, and a Qualcomm patent — that the disable is a **permanent QFPROM fuse** set during chip binning, the same mechanism that disables the (genuinely fused-out) cellular modem on this handheld. Every app-level, firmware-level, and bootloader-level route to enable CDSP was attempted and is documented here.

## Why this exists

This project went from *"the NPU is off, let's turn it on"* to a full-stack reverse-engineering campaign spanning **47 distinct work phases** — hundreds of analysis reports, well over a thousand probe captures, multiple independent kernel builds, repeated EDL/Firehose sessions, full bootloader auth reversing, a device brick and recovery, and a final on-device fuse read. It did not unlock the NPU — because there was never an unlock to find. But it produced a rigorous, reproducible answer to a question nobody had documented for this SoC.

## Where to start

<div class="grid cards" markdown>

-   :material-flag-checkered: **[The Verdict](01-the-verdict/index.md)**

    The final answer, stated plainly, with the confidence and the caveats.

-   :material-check-decagram: **[How We Proved It](02-how-we-proved-it/index.md)**

    The three independent proofs that nail it: live socinfo read, FeatureEnabler trustlet, kernel source + patent.

-   :material-format-list-checks: **[Full Test Inventory](03-methods/00-full-test-inventory.md)**

    All 47 work phases — the whole campaign, with file counts.

-   :material-folder-search: **[Methods](03-methods/index.md)**

    Deep-dive write-ups of every front: EDL, XBL, vendor_dlkm, kernel, FeatureEnabler, donors.

-   :material-file-document-multiple: **[Evidence](04-evidence/index.md)**

    Curated raw evidence — logs, on-device reads, decode reports, hashes.

-   :material-test-tube: **[Reproduce](07-reproduce/index.md)**

    Read-only steps to verify the key findings on your own Odin 3.

</div>

## The one-paragraph proof

`/sys/devices/soc0/nsp` is populated from the Qualcomm socinfo **`subset_parts`** (a.k.a. `defective_parts` / "partial-goods") table, which boot firmware fills from **QFPROM** at boot. On this device it reads `0xff` (absent) for `nsp` — the *same table, same encoding* that reports `modem = 0xff`, a known-true hardware bin-out on this no-cellular handheld. The signed XBL bootloader derives this by reading fuse word `0x221c2420`; the FeatureEnabler trustlet independently checks `qsee_is_sw_fuse_blown(fuse_id = 22 = NSP_DISABLE)`. QFPROM fuses are one-time-programmable and read-only at boot (Qualcomm patent **US 12,061,855**, "Functional circuit block harvesting"). The fuse aperture holding the NSP bit is **not** mapped to Android (`/proc/iomem` shows only the `0x221c8000` window) and `/dev/mem` is compiled out — so it cannot be altered from the OS even with root. Three independent methods agree.

!!! note "What is and isn't here"
    This site documents **original analysis, methodology, scripts, and curated text evidence**. It does **not** redistribute AYN's proprietary factory firmware or third-party donor firmware — those are documented and hashed, never published. Device serials, IMEIs, keys, and personal paths are scrubbed. See the repo's `SENSITIVE.md` for details.

---

**Status:** `CASE CLOSED` · `NSP_HARDWARE_FUSE_DISABLED` · `~94% PERMANENT` · `NOT UNLOCKABLE ON THIS DEVICE`
