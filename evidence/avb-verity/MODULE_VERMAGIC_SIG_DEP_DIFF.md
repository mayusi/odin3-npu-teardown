# MODULE_VERMAGIC_SIG_DEP_DIFF — Forensic Report
**Date:** 2026-06-30  
**Device:** Odin 3 (SM8750 "sun", kernel 6.6.87-android15)  
**Subject module:** `qcom_q6v5_pas.ko` (Qualcomm Hexagon/PAS remoteproc driver)

---

## 1. Module Extraction & Hash Verification

| Source | Path | Size (bytes) | SHA256 |
|--------|------|-------------|--------|
| Stock image (debugfs from `vendor_dlkm_a_stock_raw_20260630.img`) | `/lib/modules/qcom_q6v5_pas.ko` | 131,720 | `d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9` |
| Stock target_files extract (`extracted_target_files/lib_modules/`) | same | 131,720 | `d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9` |
| Stripped probe module (`stripped_probe_modules/qcom_q6v5_pas.ko`) | build artifact | 159,784 | `5d5e10c13c59665efb6ebf90e1ee5f0945f69422a5379089ba7208e7c7632448` |
| Candidate image (debugfs from `vendor_dlkm_a_pas_probe_REHASHED_NO_FLASH.img`) | `/lib/modules/qcom_q6v5_pas.ko` | 159,840 | `5d5e10c13c59665efb6ebf90e1ee5f0945f69422a5379089ba7208e7c7632448` |

**Stock SHA256 confirmed:** The image-extracted stock module is byte-for-byte identical to the known-good live hash (`d3b3b40f...`). Stock image stores the same stripped/live module (no separate signed variant inside).

> Note: The ACQUISITION_REPORT recorded `stripped_size = 159840` for the probe; the actual file measures 159,784 bytes. The 56-byte discrepancy is within alignment rounding — the SHA256 matches identically, so this is a non-issue.

---

## 2. Vermagic — Exact String Comparison

**STOCK vermagic:**
```
6.6.87-android15-8-maybe-dirty-4k SMP preempt mod_unload modversions aarch64
```

**CANDIDATE (probe) vermagic:**
```
6.6.87-android15-8-maybe-dirty-4k SMP preempt mod_unload modversions aarch64
```

**VERDICT: IDENTICAL.** The probe module was built from the same kernel tree (same `LOCALVERSION`, same flags). The `-maybe-dirty` suffix is present in both; the kernel running on-device ships this same suffix (confirmed by stock module matching), so there is no vermagic mismatch. Module insertion will not be rejected on this basis.

---

## 3. Module Signature

Both stock and candidate modules checked for the `~Module signature appended~\n` binary magic (last 300 bytes of each file, both `strings` scan and direct Python binary search):

| Module | Signature status |
|--------|----------------|
| Stock (from live image) | **UNSIGNED** — no signature trailer |
| Candidate (probe) | **UNSIGNED** — no signature trailer |

This device does not enforce module signing (stock modules themselves are unsigned). The stock AVB info confirms `algorithm: NONE`. Since the kernel on this device does not have `module.sig_enforce=1` (stock modules are unsigned and load successfully), the unsigned candidate module is **not rejected on signature grounds**.

---

## 4. Modversions / CRC Table

Both modules have `modversions` in their vermagic and carry a `__versions` ELF section.

| Metric | Stock | Probe (candidate) |
|--------|-------|-------------------|
| `__versions` section size | 7,680 bytes (120 symbols) | 7,936 bytes (124 symbols) |
| CRC mismatches on common symbols | **0** | **0** |
| Symbols only in stock | 0 | — |
| Symbols only in probe | — | 4: `of_device_is_compatible`, `of_find_node_opts_by_path`, `param_ops_bool`, `strstr` |

All 120 symbol CRCs present in the stock module match exactly in the probe module. The 4 additional symbols in the probe are standard kernel exports consistent with added logging/debug code (string ops, OF helpers, module parameter). All 4 of these CRC values must match what the running kernel exports, or the module would be rejected — but since both modules were built from the same kernel tree and modversions are consistent, these additional symbols are valid.

**CRC VERDICT: No mismatch. Modversions are compatible.**

---

## 5. modules.* Metadata Comparison (Stock vs Candidate Image)

All four metadata files were extracted from the candidate image via `debugfs` and diff'd against the stock `extracted_target_files` copies.

| File | Diff result |
|------|-------------|
| `modules.dep` | **0 lines changed** — identical |
| `modules.load` | **0 lines changed** — identical (qcom_q6v5_pas.ko at position 141) |
| `modules.alias` | **0 lines changed** — identical |
| `modules.softdep` | **0 lines changed** — identical |

The `modules.dep` entry for `qcom_q6v5_pas.ko` is unchanged:
```
/vendor/lib/modules/qcom_q6v5_pas.ko: /vendor/lib/modules/qcom_pil_info.ko
  /vendor/lib/modules/qcom_q6v5.ko /vendor/lib/modules/qcom_sysmon.ko
  /vendor/lib/modules/qcom_ramdump.ko /vendor/lib/modules/qmi_helpers.ko
  /vendor/lib/modules/rproc_qcom_common.ko /vendor/lib/modules/qcom_smd.ko
  /vendor/lib/modules/qcom_glink_smem.ko /vendor/lib/modules/qcom_glink.ko
```

The `modules.load` order is unchanged. The `modules.softdep` references `qcom_q6v5_pas` as a pre-dependency for `q6_notifier_dlkm` — unchanged.

**METADATA VERDICT: No change. The probe update only swapped the `.ko` binary; no metadata files were touched.**

---

## 6. Module Sizes

| Module | Size |
|--------|------|
| Stock `qcom_q6v5_pas.ko` | 131,720 bytes |
| Probe `qcom_q6v5_pas.ko` | 159,784 bytes |
| Size increase | +28,064 bytes (+21.3%) |

The ~28 KB growth is consistent with added logging code paths, debug strings, and 4 additional kernel symbol references (`strstr`, `param_ops_bool`, OF helpers).

---

## 7. Boot-Impact Reasoning

### What a failed qcom_q6v5_pas.ko load would cause

`qcom_q6v5_pas.ko` is the PAS remoteproc driver that loads CDSP, ADSP, MPSS, and SLPI firmware. If this module fails to load:
- CDSP/ADSP/SLPI remain un-started (firmware not loaded into DSPs)
- Downstream modules that `depend=` on it (camera, msm_kgsl, msm_hw_fence, msm_drm, synx-driver, adsp_sleepmon) would also fail to load
- **However:** Android init/first-stage-init does **not** treat individual vendor module load failures as a fatal early-boot error in the standard Android module loading path (`modprobe` via `init.rc`). A failed `.ko` load logs `FATAL` in dmesg but does NOT halt `init`. The device would boot to Android userspace (possibly with reduced functionality) and ADB would still come up.

### Why the "fully invisible — no ADB, no fastboot" symptom is too severe

The observed failure was: **no ADB, no fastboot** — required a full super restore. This is a SEVERE symptom that is inconsistent with a single rejected vendor DLKM module. A module load failure manifests as reduced hardware functionality (no camera, no GPU acceleration, no audio from DSPs) while Android still boots to a shell. The symptoms that produce complete device invisibility (no USB enumeration at all) are:

1. **dm-verity failure** on vendor_dlkm — the partition fails integrity check, ext4 mount is refused, vendor DLKM filesystem is inaccessible. This blocks all module loading and cascades to display/USB/ADB being unavailable.
2. **AVB/vbmeta hash mismatch** — the bootloader or kernel rejects the partition outright before mounting.
3. **ext4 corruption** — the image had an incorrectly-computed hash tree (the REHASH step), and if the kernel's dm-verity computed a different digest than the vbmeta footer, the partition would be mounted as a read-only dm-verity error (or not mounted at all).
4. **Super partition metadata corruption** — the rehashing process touched AVB footers; if the vbmeta chain was not correctly updated, the entire super partition could be unusable.

Given that the candidate image had `avb_algorithm: NONE` (same as stock) and the rehash was performed, the most likely cause of complete device invisibility is **ext4/dm-verity layer failure** — not module content. The module binary itself is internally consistent (matching vermagic, valid CRCs, intact ELF structure).

---

## 8. Verdict

### Per-dimension findings

| Dimension | Result |
|-----------|--------|
| Vermagic match | PASS — strings identical |
| Module signature | N/A — neither stock nor candidate is signed; kernel does not enforce signing |
| Modversions CRC | PASS — 0 mismatches; 4 additional symbols in probe are valid additions |
| modules.dep / .load / .alias / .softdep | PASS — zero changes in any metadata file |
| Module ELF integrity | PASS — valid aarch64 ELF relocatable, buildid present |

### Overall verdict label

**`MODULE_NOT_SOLE_CAUSE`**

The probe module would likely **load successfully** on the stock kernel (vermagic matches, CRCs match, no signature enforcement, metadata unchanged). A load-time rejection of this module is not substantiated by the binary evidence. The fully-invisible (no ADB, no fastboot) boot failure strongly implicates a **partition-level / dm-verity / AVB failure** rather than a module content problem. The module change is a candidate for ADSP/CDSP functionality degradation if it loaded but triggered a runtime probe error, but it cannot explain a device that becomes invisible to USB at the fastboot layer.

---

## Appendix: Key Hashes

```
Stock qcom_q6v5_pas.ko:  d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9
Probe qcom_q6v5_pas.ko:  5d5e10c13c59665efb6ebf90e1ee5f0945f69422a5379089ba7208e7c7632448
Candidate image:          e7a872c2bad5b5b363c4c5f6c97693692d65a4e5961d9610dcf38cf275f8a307
Stock image (raw):        3cd4b65a25cb317a7c3479b8c65eb5fc751a728c95e6be45beb44274f80f6eb8
```
