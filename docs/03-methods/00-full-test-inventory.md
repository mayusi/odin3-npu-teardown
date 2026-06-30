# Full Test Inventory — every phase, every probe

> This page exists to make one thing unambiguous: the conclusion that the NPU is fused off is **not** the result of a single test or a single afternoon. It is the residue of a sustained, multi-front reverse-engineering campaign — **47 distinct work phases**, spanning app-layer probing, firmware reverse-engineering, multiple kernel builds, EDL/Firehose forensics, bootloader auth analysis, TrustZone probing, donor-firmware teardowns, live boot experiments (including one that bricked the device and a full recovery), and finally an on-device fuse read.

## By the numbers

| Metric | Count |
|---|---|
| Distinct work phases (tracked directories) | **47** |
| Written analysis reports (`.md`) | **199** |
| Structured result files (`.json`) | **177** |
| Probe / log / capture files (`.txt`) | **1,397** |
| Scripts authored (probes, runners, decoders) | **23+** |
| Kernel/module patches written | **7+** |
| Independent kernel build / source efforts | **5** |
| Firmware images analyzed (factory + donor) | dozens (hashed, never redistributed) |
| Independent confirmations of the final verdict | **3** (see [How We Proved It](../02-how-we-proved-it/index.md)) |

> **The "three proofs" are only the closing argument.** The [How We Proved It](../02-how-we-proved-it/index.md) tab presents the three *independent* methods that nail the final verdict. Everything below is the campaign that led there — the dozens of avenues opened, tested, and closed before the fuse read settled it.

## Every phase

The investigation is organized into discrete phases, each with its own captured evidence. Grouped by front:

### App layer — can userspace reach the NPU?
| Phase | What it did | Result |
|---|---|---|
| `npubench-npu-research` / `npubench-npu-try` | NPUBench / QNN / LiteRT NPU-execution attempts | Blocked — no CDSP |
| `npubench-no-root-probe` | Headless no-root QNN delegate probe | `NPU_BLOCKED_PLATFORM` — QNN loads `libcdsprpc.so`, fails at FastRPC domain 3 |
| `remaining-paths-20260626` | Inventory of every conceivable route | Scoped the whole campaign |
| `odin-settings-inspect` / `odin-root-runner` | Device inspection + a full root-script-runner automation harness | Enabled read-only root probing |

### Firmware surface — is the firmware even present?
| Phase | What it did | Result |
|---|---|---|
| `cdsp-firmware-crosscheck-20260628` | Cross-checked CDSP firmware presence | Present but policy-blocked |
| `cdsp_enable` | CDSP-enable attempt surface | Blocked upstream |
| `trustzone-cdsp-probe-20260628` | TrustZone / QSEE CDSP path probing | CDSP gated before TZ bringup |
| `qfprom-qspa-crosscheck-20260628` | QFPROM vs QSPA policy cross-check | Located the `0x221c2420` input |

### Bootloader / XBL — where is the switch, and is it reachable?
| Phase | What it did | Result |
|---|---|---|
| `aime-npu-unlock-20260628` | **AArch64 string-reference scan** of XBL + offline decode (895 files) | Found the `feature_word[15] |= read32(0x221c2420) & 0xff` rule |
| `xbl-auth-review-20260628` | XBL authentication / signature review | SecBoot SHA-384 — unsignable |
| `xbl-enforcement-20260628` | XBL enforcement analysis + recovery-first plan | Hash-covered, high risk |
| `xbl-config-report-crosscheck-20260628` | xbl_config RetailImages cross-check | CDSP entries defined, selection gated |
| `xbl-offline-certainty-20260628` | Final offline go/no-go on an xbl_config CDSP patch | Hash-covered, falsified as a route |
| `xbl_patch` | Built a structural xbl_config CDSP/NPU enablement patch | Offline only — cannot be signed |
| `xbl-next-paths-20260628` | **CDSP/NPU escalation packet** + NSP source recon (1,024 files) | Mapped every remaining lane |
| `download-vetting-20260628` | Vetting of downloaded firmware/tools | Inputs validated |

### EDL / Firehose — can we read the fuse directly?
| Phase | What it did | Result |
|---|---|---|
| `edl-readonly-proof-20260628` | Proved read-only EDL access | Odin enumerates as Qualcomm 9008 |
| `edl-deep-vetting-20260628` | Deep Firehose loader vetting | Loader works for storage, **no `peek`** |
| `edl-source-vetting-20260628` | Firehose loader source vetting | No `peek`-capable loader found |
| `edl-bootcritical-backup-20260628` | **Read-only boot-critical partition backups** | Full safety net captured |
| `odin3-edl-rescue-package` / `odin3-rescue-share` | Built EDL rescue packages | Recovery readiness |

### Kernel — can a custom kernel override the policy?
| Phase | What it did | Result |
|---|---|---|
| `android_kernel_ayn_cq8725s` | AYN CQ8725S kernel source (full tree, 94 files) | Build anchor |
| `kernel-cq8725s-nocheckout` | Kernel source staging (207 files) | Source prepared |
| `aime-kernel-compat-20260629` | **6.6.87 marker / compat build probes (510 files)** | Vermagic/compat mapped |
| `aime-kernel-bypass-20260629` | CDSP function map + bypass analysis | CDSP binds *after* policy gate |
| `aime-kernel-last-mile-20260629` | Active DT CDSP map + boot-policy trace | `BUILD_READY_NO_FLASH` |
| `aime-module-source-map-20260630` | Runtime module source map | Mapped `qcom_q6v5_pas` et al. |
| `aime-enthusiast-last-mile-20260629` | **Log-only CDSP probe patch** (compiled) | Patches built, not flashable |
| `aime-npu-unlocker-finish-20260629` | CDSP probe build report + probe guide | Build verified |

### Live boot experiments — and the brick
| Phase | What it did | Result |
|---|---|---|
| `aime-boot-a-logonly-probe-20260629` | One-boot `boot_a` log-only probe | Probe-risk confirmed |
| `aime-recovery-checkpoint-20260629` | Recovery checkpoint before live probes (111 files) | Rollback prepared |
| `aime-vendorboot-oneboot-probe-20260630` / `-default-on-` | `vendor_boot` one-boot probes | Did not bring up CDSP |
| `aime-vendor-dlkm-acquire-20260630` | Acquired + prepared a `vendor_dlkm` candidate | `READY_NO_FLASH` |
| `aime-vendor-dlkm-oneboot-probe-20260630` | **The `vendor_dlkm` flash that bricked the device** (337 files) | Device went fully invisible |
| `aime-recovery-before-vendorboot-test-20260630` | Pre-test recovery state capture (392 files) | Baseline preserved |
| `aime-fastboot-recovery-20260630` / `black-screen-recovery` | **Recovery from the brick** | Full stock `super` restore → recovered |
| `aime-vendor-dlkm-postmortem-20260630` | **Offline root-cause of the brick** | `AVB_METADATA_BROKEN` (signed-vbmeta verity mismatch) |

### Donor firmware — does the same SoC enable CDSP elsewhere?
| Phase | What it did | Result |
|---|---|---|
| `aime-npu-unlock-20260629` | Donor comparison: Realme GT7 Pro, Xiaomi 15 Ultra, iQOO 13, OnePlus 13 (51 files) | All run CDSP; configs device-locked |

### The closeout — factory firmware + the fuse read
| Phase | What it did | Result |
|---|---|---|
| `odin3-folders-vet-20260630` | Vetted the AYN factory firmware package | Authentic factory set |
| `aime-npu-feasibility-20260630` | **FeatureEnabler decode + 5-lane feasibility sweep + adversarial verify** | `featenabler` licenses display/video only |
| `aime-fuse-read-20260630` | **On-device read-only fuse / socinfo read** | `nsp=0xff` via QFPROM subset-parts → **hardware fuse** |

## What this inventory proves

Every credible avenue to enable CDSP — app, firmware, bootloader, kernel, EDL, donor transplant, live patching — was opened and closed with evidence. The verdict isn't an assumption that survived because nothing disproved it; it's the **last hypothesis standing after dozens were actively eliminated**, and then independently confirmed three ways. The full evidence trail lives in [`evidence/`](../04-evidence/index.md) and the source artifacts behind every row above.
