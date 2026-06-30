# Overview: Why the Odin 3 NPU Is Gone Forever

> **TL;DR** The Hexagon Neural Processing Unit (NPU) on the AYN Odin 3 is permanently disabled — not by software, not by AYN, but by a one-time-programmable hardware fuse blown at the Qualcomm factory. This investigation proves it beyond reasonable doubt.

---

## The Headline Result

The AYN Odin 3 runs a Qualcomm **CQ8725S** SoC — a "Dragonwing" IoT/industrial bin of the SM8750 ("sun") platform. This chip reached the factory with its **NSP (Neural Signal Processor) fuse group blown**, marking the entire neural processing subsystem as absent. Every layer of firmware and kernel software above that fuse reads the same answer and acts on it: the NSP is not here, don't try.

The result is final. There is no firmware update, no AYN software unlock, no rooted-device workaround that can unburn a hardware fuse. Running QNN/HTP workloads on this device's on-chip NPU is **not possible** — not today, not with any future OS release.

---

## What This Investigation Is

This is a systematic reverse-engineering audit of the complete NPU enable path on the Odin 3, from silicon fuse through bootloader, kernel, and Android userspace. It was conducted on a **bootloader-unlocked device** (`verifiedbootstate=orange`) with full root access.

The work covers:

- Reading live SoC identity registers via `/sys/devices/soc0`
- Reverse-engineering the XBL (eXtensible Bootloader) fuse-read path
- Auditing the Qualcomm QSPA (SoC Platform Arbiter) and FeatureEnabler trustlets
- Tracing the CDSP (Compute DSP) bring-up chain from device-tree through remoteproc
- Verifying that no CDSP firmware image is loaded
- Cross-referencing against Qualcomm QFPROM documentation and patent literature

The conclusion carries **~94% confidence** that the block is a permanent OTP hardware fuse. The remaining ~6% reflects one physically unreadable register — the fuse word itself — which cannot be confirmed on-device because the QFPROM aperture mapped to HLOS does not include that address.

---

## Start Here

If you want to understand **what** happened (the verdict), go to:

- [`01-the-verdict/`](../01-the-verdict/index.md) — the definitive answer, the full causal chain, and the confidence breakdown

If you want to understand **the hardware** context:

- [`00-overview/hardware.md`](hardware.md) — the Odin 3, the CQ8725S SKU, and what "binning" means
- [`00-overview/npu-stack.md`](npu-stack.md) — how the Hexagon NPU, CDSP, FastRPC, and QNN fit together

If you want to understand **how to read this repo**:

- [`00-overview/how-to-read.md`](how-to-read.md) — evidence structure, what's included/excluded, and the confidence framework

If you want the **proof**:

- [`02-how-we-proved-it/`](../02-how-we-proved-it/) — step-by-step methodology
- [`04-evidence/`](../04-evidence/) — raw artifacts, dumps, and annotated output

---

## Doc Map

```
docs/
├── 00-overview/          ← You are here
│   ├── index.md          — This page: orientation and result summary
│   ├── hardware.md       — The Odin 3 and CQ8725S SoC context
│   ├── npu-stack.md      — The NPU software/firmware stack explained
│   └── how-to-read.md    — Repo organization and evidence framing
│
├── 01-the-verdict/       ← Start here if you want the answer
│   └── index.md          — The definitive verdict and causal chain
│
├── 02-how-we-proved-it/  — Investigation methodology, step by step
├── 03-methods/           — Tools and techniques reference
├── 04-evidence/          — Raw artifacts and annotated dumps
├── 05-timeline/          — Chronological investigation log
├── 06-faq/               — Common questions and misconceptions
└── 07-reproduce/         — How to reproduce the key checks yourself
```

---

## A Note on Scope

This repo documents **why the NPU cannot be enabled**. It does not document general Odin 3 rooting, general SM8750 platform research, or QNN application development. Those topics are out of scope.

The investigation is **complete**. This repo is a record, not an ongoing project.
