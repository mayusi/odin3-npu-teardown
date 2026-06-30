<div align="center">

# Odin 3 / Snapdragon CQ8725S — NPU (NSP) Unlock Investigation

### A complete, evidence-backed teardown of *why* the Hexagon NPU on the AYN Odin 3 cannot be enabled — and everything that was tried to enable it.

**Device:** AYN Odin 3 · **SoC:** Qualcomm `sun` / SM8750-class, model **CQ8725S-3-AA** (Dragonwing / IoT bin)
**Verdict:** `NSP_HARDWARE_FUSE_DISABLED` — the NPU subsystem is **fused off at the factory**, the same way the modem is. **Not unlockable by software, firmware, signing, or any on-device path (~94% confidence).**

</div>

---

> **TL;DR.** The Odin 3 ships with `/sys/devices/soc0/nsp = 0xff` (NSP subsystem absent) while `npu = 0x0` (NPU compute IP present). This repo proves — with on-device reads, firmware reverse-engineering, kernel-source analysis, and a Qualcomm patent — that the disable is a **permanent QFPROM fuse** set during chip binning, identical in mechanism to the (genuinely fused-out) modem on this cellular-less handheld. Every app-level, firmware-level, and bootloader-level route to enable CDSP was attempted and is documented here.

## Why this exists

This project went from *"the NPU is off, let's turn it on"* to a full-stack reverse-engineering campaign spanning **47 distinct work phases** — hundreds of analysis reports, well over a thousand probe captures, multiple independent kernel builds, repeated EDL/Firehose sessions, full bootloader auth reversing, a device brick and recovery, and a final on-device fuse read. The fronts: no-root QNN probing → firmware surface inventory → XBL/boot-policy analysis → EDL/Firehose storage forensics → donor-firmware comparison → kernel source builds (×5) → live `vendor_dlkm`/`vendor_boot` boot probes (one of which **bricked the device and was fully recovered**) → factory-firmware reverse-engineering → a final **read-only on-device fuse read** that closed the case. The [Full Test Inventory](docs/03-methods/00-full-test-inventory.md) catalogs every phase.

It did not unlock the NPU — because there was never an unlock to find. But it produced a rigorous, reproducible answer to a question nobody had documented for this SoC. That's what's published here.

## Read the docs

| Tab | What's in it |
|---|---|
| **[Overview](docs/00-overview/)** | What the device is, what the NPU/NSP/CDSP/FastRPC stack is, and how to read this repo |
| **[The Verdict](docs/01-the-verdict/)** | The final answer, stated plainly, with the confidence and the caveats |
| **[How We Proved It](docs/02-how-we-proved-it/)** | The three independent proofs: live socinfo read, FeatureEnabler trustlet, kernel source + patent |
| **[Methods](docs/03-methods/)** | Every avenue tried — EDL/Firehose, XBL/boot policy, vendor_dlkm, kernel build, FeatureEnabler, donors |
| **[Evidence](docs/04-evidence/)** | Index of curated raw evidence (logs, reads, reports, hashes) in [`/evidence`](evidence/) |
| **[Timeline](docs/05-timeline/)** | Chronological log of every phase and result |
| **[FAQ](docs/06-faq/)** | "Can't AYN just enable it?", "Is it really permanent?", and more |
| **[Reproduce](docs/07-reproduce/)** | Read-only steps to verify the key findings on your own Odin 3 |

## The one-paragraph proof

`/sys/devices/soc0/nsp` is populated from the Qualcomm socinfo **`subset_parts`** (a.k.a. `defective_parts` / "partial-goods") table, which boot firmware fills from **QFPROM** at boot. On this device it reads `0xff` (absent) for `nsp` — the *same table, same encoding* that reports `modem = 0xff`, a known-true hardware bin-out on this no-cellular handheld. The signed XBL bootloader derives this by reading fuse word `0x221c2420`; the FeatureEnabler trustlet independently checks `qsee_is_sw_fuse_blown(fuse_id = 22 = NSP_DISABLE)`. QFPROM fuses are one-time-programmable and read-only at boot (Qualcomm patent **US 12,061,855**, "Functional circuit block harvesting"). The fuse aperture holding the NSP bit is **not** mapped to Android (`/proc/iomem` shows only the `0x221c8000` window) and `/dev/mem` is compiled out — so it cannot be altered from the OS even with root. Three independent methods agree.

## ⚠️ What is and isn't in this repo

This repo contains **original analysis, documentation, scripts, and curated text evidence** (logs, on-device reads, decode reports, hashes). It **does not** redistribute:
- **AYN's proprietary factory firmware** (`ODIN3_FOLDERS` and its signed images) — documented and hashed, never published.
- **Third-party donor firmware** binaries used for comparison — referenced by source, not included.
- Any device serials, IMEIs, keys, or personal paths — scrubbed (see [`SENSITIVE.md`](SENSITIVE.md)).

See [`docs/04-evidence`](docs/04-evidence/) for what *is* included and why.

## Status

`CASE CLOSED` · `NSP_HARDWARE_FUSE_DISABLED` · `~94% PERMANENT` · `NOT UNLOCKABLE ON THIS DEVICE`

## License

Documentation and original analysis: see [`LICENSE`](LICENSE). Quoted firmware strings, register names, and third-party identifiers remain property of their respective owners and are reproduced here under fair-use for research/interoperability documentation.
