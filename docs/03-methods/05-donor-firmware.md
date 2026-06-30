# 05 · Donor Firmware Analysis

> **Status:** `EXACT_ODIN_NSP_RULE_PRESENT` confirmed across 11 SM8750 donor files.
> Donor feature configs are device-identity-locked and cannot be transplanted.
> No CDSP PIL firmware candidates found. All loaders rejected at runtime.

---

## Why Donors

The hypothesis: if every other Snapdragon 8 Elite phone ships with the NPU/CDSP
active, their firmware must contain the policy and firmware blobs that enable it.
Extract those, understand what differs, and you have the shape of what the Odin 3
is missing — or at minimum, proof that the same XBL silicon carries the capability.

The donor sweep ran across six OEM firmware packages spanning four device families,
all on the SM8750 "sun" die.

---

## Donor Matrix

| Source | Device | Trust | Locally extracted |
|--------|--------|-------|-------------------|
| Realme GT 7 Pro (RMX5011) | SM8750 / sun | HIGH | Yes — full image set |
| Xiaomi 15 Ultra (xuanyuan) | SM8750 / sun | HIGH | Yes — full image set |
| iQOO 13 | SM8750 / sun | HIGH | Source identified, not locally packaged |
| OnePlus 13 (community OTA) | SM8750 / sun | HIGH | Source identified, not locally packaged |
| REDMAGIC 10 Pro (NX789J) | SM8750 / sun | HIGH | Source identified, not locally packaged |
| Xiaomi 15 (dada) | SM8750 / sun | HIGH | Source identified, not locally packaged |

> All donors confirmed SM8750 "sun" via compatible strings and SoC identifiers.
> All run CDSP actively. None are IoT-binned.

---

## The Key Finding: `EXACT_ODIN_NSP_RULE_PRESENT`

Every donor XBL and every SM8750 Firehose programmer tested carries **the exact same
NSP-disable rule** that is present in the Odin 3's own XBL:

| File (donor source) | Verdict | Rule offsets |
|---------------------|---------|-------------|
| OnePlus SM8750 Chimera loader | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x101f7c`, `0x1612c4` |
| REDMAGIC/Nubia NX789J | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x102c94`, `0x1658d4` |
| Realme GT 7 Pro `prog_firehose_ddr.elf` | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x102d14`, `0x16283c` |
| Realme GT 7 Pro `xbl_s.melf` | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x103f54` |
| SM8750 Iqinix candidate | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x1041b4`, `0x16b8b4` |
| Xiaomi 15 Ultra `xbl_s_devprg_ns.melf` | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x108354`, `0x16fbe4` |
| iQOO 13 (two builds) | `EXACT_ODIN_NSP_RULE_PRESENT` | `0x105034`, `0x17248c` |

The rule itself: `value = read32(0x221c2420) & 0xff; feature_word[15] |= value`
(see `evidence/npu-unlock-20260628/AIME_NPU_UNLOCK_CURRENT_STATE_20260628.md:47–67`).

**What this means:** the NSP gate is baked into the Qualcomm BSP XBL shipped to
every SM8750 OEM. The rule is present in the Odin's own XBL with the same opcodes.
The difference is not the *rule* — it is the **QFPROM input at `0x221c2420`**.
On donor phones that input reads `0x00`; on the Odin 3 it reads `0xff`. That single
byte is what separates an active NPU from a disabled one.

---

## Loader Vetting: All Rejected

Seven SM8750 Firehose loaders were evaluated to see if any could `peek` at
`0x221c2420` from EDL mode.

| Loader | Source | Static peek string | Runtime result |
|--------|--------|--------------------|----------------|
| `xuanyuan … xbl_s_devprg_ns.melf` | Xiaomi 15 Ultra | Yes | `LOADER_RUNTIME_NO_PEEK` (test-signed; see §09) |
| `diablosat__zte_NX789J.elf` | REDMAGIC/Nubia | Yes | `LIVE_REJECT_NO_PEEK` → retired |
| `Iqinix … SNAPDRAGON 8 ELITE.melf` | SM8750 candidate | Yes | `LIVE_REJECT_NO_PEEK` → retired |
| `iqoo_13.melf` (and redownload) | iQOO 13 | Yes | `LIVE_REJECT_NO_PEEK` → retired |
| `OPPO SM8750 Chimera V1.0.05.melf` | OnePlus/OPlus | No | `STATIC_REJECT_NO_PEEK_STRING` |
| `prog_firehose_ddr.elf` | Realme GT 7 Pro | No | `STATIC_REJECT_NO_PEEK_STRING` |

No loader that statically advertises `peek` passed at runtime. All SM8750 retail
Firehose programmers that survive static screening are rejected by the device's PBL —
because they carry test-key signatures incompatible with the production OEM cert chain.
See [Method 09 — FeatureEnabler & Firehose dead-end](09-featenabler-decode.md) for
the full signing chain analysis.

---

## Policy Carrier Survey

The donor sweep also catalogued every partition that references NSP/CDSP policy terms,
to understand which firmware components might carry the enabling configuration:

| Partition | Key terms found | Verdict |
|-----------|----------------|---------|
| `vendor_boot.img` (Realme, Xiaomi) | `feature_id`, `qfprom`, `nsp`, `cdsp`, `qspa`, `rpmb` | `DIRECT_TARGET_PRESENT_INSPECT_ONLY` |
| `devcfg.mbn` | `qfprom`, `nsp`, `rpmb` | Inspect only |
| `dtbo.img` | `swfuse`, `cdsp` | Inspect only |
| `featenabler.mbn` | `featureenabler`, `feature_id`, `configure_swfuse`, `swfuse`, `rpmb` | Inspect only — **does not gate NSP** (see §09) |
| `xbl_config.elf` | `nsp`, `cdsp` | Inspect only |
| `aop.mbn` | `nsp`, `cdsp` | Inspect only |

All are classified `INSPECT_ONLY` — they *carry* policy terms but transplanting
donor policy partitions to the Odin 3 is not viable because:

1. These partitions are signed by OEM keys specific to each vendor.
2. ABL/XBL policy readers on the Odin 3 would reject foreign-signed images.
3. Even if signatures could be bypassed, the QFPROM fuse value at `0x221c2420`
   is read before any of these partitions are consulted — the NSP gate fires in
   XBL, before Linux, before vendor_boot, before FeatureEnabler.

---

## XBL Direct Bypass: Hash-Covered

The one-byte bypass at offset `0x10935c` in the Odin's own XBL would, in theory,
patch the NSP rule in-place. The analysis verdict:

- `CANDIDATE_PRECISE_BUT_HASH_COVERED`
- Mask offset: `0x10935c`
- Patched hash matches stored hash: **False**

The XBL image is hash-verified by the PBL. Any byte modification invalidates the
measurement, and the PBL will refuse to boot a tampered XBL on a retail device
with secure boot enabled. This path requires either an engineering-fused device or
a signed XBL from AYN/Qualcomm.

---

## Donor CDSP Firmware

No CDSP firmware candidates (`cdsp.mdt`, `cdsp_dtb.mdt`) were extracted from any
donor package that could serve as a transplant. The firmware names are present in
the Odin 3's own DT node (`firmware-name = "cdsp.mdt, cdsp_dtb.mdt"`) confirming
the kernel knows where to load it — but the CDSP remoteproc node has `status = "no"`,
so the firmware is never requested. See [Method 06 — Kernel Build](06-kernel-build.md).

---

## Conclusion

The donor sweep established three facts that anchor every subsequent method:

1. **The hardware capability is present.** Every SM8750 donor phone runs CDSP/NPU
   on the same die with the same XBL rule. The Odin is silicon-equivalent.

2. **The gate is a single fuse read.** `read32(0x221c2420) & 0xff` in XBL is the
   decisive input. Donor phones return `0x00`; the Odin returns `0xff`.

3. **Nothing is transplantable.** Donor policy carriers are OEM-signed and
   device-identity-locked. Donor loaders are test-signed and rejected by the Odin's
   PBL at runtime.

> **Evidence files:**
> `evidence/npu-unlock-20260629/AIME_NPU_UNLOCK_NEXT_MOVE_20260629.md`
> `evidence/npu-unlock-20260628/donor_extract_20260628/` (Realme, Xiaomi extracts)
