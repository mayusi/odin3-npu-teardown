# How to Read This Repo

This page explains how the documentation is structured, what evidence is included, what is deliberately excluded (and why), and how to interpret the confidence levels used throughout.

---

## Repository Structure

The repo has two top-level content areas:

```
TheDocumentationRepo/
├── docs/        ← Documentation pages (what you're reading now)
└── evidence/    ← Raw artifacts from the investigation
```

The `docs/` tree is organized by **conceptual stage**:

| Directory | Contents |
|---|---|
| `00-overview/` | Background, hardware context, stack explanation, this guide |
| `01-the-verdict/` | The definitive answer and full causal chain |
| `02-how-we-proved-it/` | Step-by-step investigation methodology |
| `03-methods/` | Tools, commands, and techniques reference |
| `04-evidence/` | Annotated raw artifacts |
| `05-timeline/` | Chronological investigation log |
| `06-faq/` | Common questions and misconceptions |
| `07-reproduce/` | How to reproduce the key checks on your own device |

The recommended reading order depends on what you want:

- **Just the answer:** `01-the-verdict/index.md`
- **Understand the context first:** `00-overview/hardware.md` → `00-overview/npu-stack.md` → `01-the-verdict/`
- **Want to verify the proof:** `02-how-we-proved-it/` → `04-evidence/`
- **Want to reproduce it:** `07-reproduce/`

---

## The `evidence/` Directory

The `evidence/` directory contains primary source artifacts referenced by the documentation. These are organized by investigation phase and referenced from doc pages using relative paths like `evidence/socinfo/nsp_byte.txt`.

Evidence files include:

- **Sysfs dumps** — raw text output from `/sys/devices/soc0/` nodes
- **Debugfs dumps** — firmware slot state, remoteproc state
- **Reverse-engineering output** — annotated disassembly excerpts, trustlet call traces
- **Device-tree excerpts** — relevant DT node definitions
- **Build property extracts** — `getprop` output filtered to relevant properties
- **Kernel log excerpts** — `dmesg` output showing CDSP bring-up absence

> **Note** All evidence files have been scrubbed of device serial numbers, IMEIs, and any personally identifying information before inclusion in this repo. If a serial number reference is needed for context, the placeholder `<SERIAL>` is used.

---

## What Is NOT Included (and Why)

Several categories of material are deliberately excluded from this repo:

### Qualcomm proprietary firmware blobs

The XBL bootloader binary, the FeatureEnabler trustlet binary, and the QSPA firmware image are Qualcomm proprietary. Redistributing them would violate Qualcomm's IP rights. The documentation describes what these components do, references their behavior as observed through RE, and cites specific call signatures and behavior — but does not include the binaries themselves.

The reverse-engineering was conducted legally on a legitimately owned device for research and documentation purposes.

### QFPROM raw register dumps

The QFPROM fuse word at `0x221c2420` **cannot be read on-device** — the HLOS aperture does not map it, and `/dev/mem` is compiled out. There is no artifact to include. The documentation is transparent about this gap and it is factored into the confidence framing.

### Root exploit chains or jailbreak tooling

This repo is not a rooting guide. The device was already bootloader-unlocked (`verifiedbootstate=orange`, `veritymode=eio`, slot `_a`). Methods for obtaining root access are not documented here.

### Speculative or unverified secondary sources

Every factual claim in this repo is grounded in either: (a) direct on-device observation, (b) reverse-engineering of code running on the device, (c) published Qualcomm patent or kernel source, or (d) clearly labeled inference from the above. Forum speculation, secondhand reports, and "I heard that…" claims are not included.

---

## Confidence Framework

Not all conclusions in this repo carry the same certainty. Pages and callouts use the following framework:

| Label | Meaning |
|---|---|
| **Verified** | Directly observed on-device or in RE output. Reproducible. |
| **Inferred** | Strongly implied by verified evidence; consistent with all known data. |
| **Assessed** | Judgment call based on the weight of evidence; stated as a probability estimate. |

The headline conclusion — "NSP is permanently fused off" — is **Assessed at ~94% confidence**. The gap to 100% exists because the specific QFPROM fuse bit (`0x221c2420`) cannot be read directly on-device. Every other observable in the system behaves identically to how it would behave if that fuse were blown, and no observable contradicts it.

The remaining ~6% represents the theoretical possibility that some other mechanism — not a blown fuse — could explain all observed behavior in a way that is also reversible. No plausible such mechanism has been identified.

> **Note** "94% confident" is not a hedge. It means: if you ran this investigation on 100 identical Odin 3 units, we expect 94 of them to be hardware-fused and 6 to have a different (currently unidentified) explanation. It does not mean "maybe AYN can fix it with a firmware update." The remaining uncertainty is in our measurement ability, not in the permanence of the fuse if it is indeed blown.

---

## Citing This Repo

If you reference this investigation in another context, please cite the specific evidence files and doc pages rather than paraphrasing conclusions without basis. Chains of inference become distorted quickly when paraphrased across multiple hops.
