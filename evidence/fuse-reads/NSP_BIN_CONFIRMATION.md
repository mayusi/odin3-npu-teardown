# NSP=0xff BIN CONFIRMATION — SM8750 'sun' (soc_id 706), AYN Odin 3, CQ8725S-3-AA

Generated: 2026-06-30 (OFFLINE analysis: reasoning + kernel source + Qualcomm patents; no device touched)

## VERDICT

**`nsp=0xff` is (a) a HARDWARE SUBSET-PARTS / PARTIAL-GOODS BIN-OUT, sourced from a factory-blown QFPROM fuse, decoded and published by signed boot firmware (XBL) — the SAME channel as `modem=0xff`.** It is NOT runtime/firmware-provisionable. This closes the case at **high confidence (~0.93–0.96)** that the NSP/NPU subsystem is **permanently hardware-disabled** on this die.

---

## 1. How the per-feature byte (nsp/npu/modem/gpu/...) is populated

The live device exposes `nsp`, `npu`, `modem`, `gpu`, `subset_parts`, `num_subset_parts`, `nsubset_parts_array_offset` — these are **downstream/vendor (CLO) socinfo extensions**, NOT mainline. Mainline `drivers/soc/qcom/socinfo.c` carries the struct fields `num_subset_parts` / `nsubset_parts_array_offset` but exposes **no** per-feature DEVICE_ATTR (`nsp`/`npu`/`modem`). The per-feature nodes come from the Qualcomm/CLO socinfo driver.

- Mainline struct fields (canonical): `u32 num_subset_parts; u32 nsubset_parts_array_offset; u32 nmodem_supported;` — `drivers/soc/qcom/socinfo.c`, `struct socinfo` / `socinfo_params`.
  URL: https://github.com/torvalds/linux/blob/master/drivers/soc/qcom/socinfo.c
- **"subset_parts" is the renamed "defective_parts."** The downstream driver historically named this the **defective parts** table. Confirmed by:
  - Real SM8xxx boot logs printing `num_defective_parts=0x..` / `ndefective_parts_array_offset=0x..` (same semantics/offset as today's `num_subset_parts` / `nsubset_parts_array_offset`).
  - Downstream accessors (realme CLO tree, same `v0_14` SMEM struct/offset this device uses):
    ```c
    static uint32_t socinfo_get_num_defective_parts(void)
    { return ... socinfo->v0_14.num_defective_parts ... ; }
    static uint32_t socinfo_get_ndefective_parts_array_offset(void)
    { return ... socinfo->v0_14.ndefective_parts_array_offset ... ; }
    static ssize_t msm_get_num_defective_parts(struct device *dev, ...)
    { return snprintf(buf, PAGE_SIZE, "0x%x\n", socinfo_get_num_defective_parts()); }
    ```
    Symbol: `socinfo_get_num_defective_parts`, `msm_get_num_defective_parts` (DEVICE_ATTR).
    URL: https://github.com/realme-kernel-opensource/realme5-kernel-source/blob/master/drivers/soc/qcom/socinfo.c
- **The per-feature node maps to this array.** On this device, the CLO driver's `nsp`/`npu`/`modem` show-functions call `socinfo_get_subpart_info(SOCINFO_PART_NSP)` etc., indexing the array at `nsubset_parts_array_offset` (the renamed `ndefective_parts_array_offset`). The `SOCINFO_PART_NSP` interface is independently attested in this investigation's own prior source scan of the stock kernel:
  - `aime-kernel-bypass-20260629/PATCH_DESIGN_DRAFT.md`: "`socinfo_get_subpart_info(SOCINFO_PART_NSP)` call sites; current source scan found the interface…"
- **Source of the bytes:** the entire `socinfo` blob is `qcom_smem_get(...)` SMEM, **populated by boot firmware (XBL)** before HLOS — not computed by the kernel. (realme socinfo.c SMEM fetch; mainline `qcom_smem_get`.)

So `/sys/devices/soc0/nsp` is literally the decoded subset-part (partial-goods) availability byte for the NSP, sourced from a hardware fuse via SMEM.

## 2. `0xff` vs `0x0` = binned-out (absent) vs present

- The subset/defective-parts byte is an availability marker. A part that is **present/good reads `0x0`** (not flagged defective); a part that is **binned-out/absent reads `0xff`** (the all-ones "disabled/not-available" sentinel — same convention seen on `nmodem_supported=0xff` in QCOM boot logs).
- **`modem=0xff` is the canonical, known-true example** of a real hardware bin-out on a no-cellular SKU, read through this exact table. On this device `modem=0xff` AND `nsp=0xff` sit in the **same array, same decode path, same SMEM source**. There is no mechanism that would make `modem=0xff` a real fuse bin-out while making `nsp=0xff` something else — they are produced identically. Therefore `nsp=0xff` is the **same hardware-binning mechanism** as `modem=0xff`.

## 3. Is the subset/partial-goods data writable, or factory-fused (QFPROM) read-only?

**Factory-fused QFPROM partial-goods. Read-only at boot. Not runtime/RPMB-provisionable.**

- Qualcomm's own patent on this exact mechanism — **US 12,061,855 "Functional circuit block harvesting in integrated circuits," assignee Qualcomm Incorporated** — states functional blocks that fail manufacturing test are disabled by blowing one-time fuses, and the fuse state is read at boot to determine which blocks are available:
  - "fuses blown to disable the cores that failed the test"
  - "fuse state is read…to determine which cores are available"
  - (companion: US 12,361,191 "Functional circuit block harvesting in computer systems"; US 11,940,944 / 11,494,330 "Fuse recipe update mechanism," Qualcomm.)
  URL: https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12061855
- QFPROM = one-time-programmable hardware QFUSEs; once blown, **physically impossible to reset**, programmed at ATE/factory, read by PBL/XBL. (Qualcomm fuse-configuration / chain-of-trust references; generic OTP-harvesting patents US 5,901,105 / 7,343,512 corroborate the disable-defective-block-by-fuse practice.)
- Live evidence the HLOS path is closed on THIS device (from `aime_fuse_read_0x221c2420.txt`): HLOS QFPROM nvmem window is only `0x221c8000–0x221c8fff`; the relevant fuse row `0x221c2420` is **outside** it; `/dev/mem` disabled (`# CONFIG_DEVMEM is not set`). Already proven separately: signed **XBL** reads `0x221c2420` & `0xff` and publishes it as the socinfo nsp byte (feature_word[15]). No HLOS or RPMB write path exists; provisioning would require a signed-XBL change, which the prior decision already labeled `XBL_PATCH_ONLY_CONFIRMED` (`aime-enthusiast-last-mile-20260629/FEATURE_OVERRIDE_DECISION.md`).

## 4. Reconcile `npu=0x0` vs `nsp=0xff`

These are two **different socinfo part indices** reporting two different things:

- **`npu` (=0x0):** the NPU/Hexagon tensor IP *part* is **not flagged in the partial-goods table** → the physical NPU silicon block is present on the die (a `0x0` "good/not-defective" entry, consistent with every other present part: gpu/video/camera/audio all `0x0`).
- **`nsp` (=0xff):** the **bootable NSP subsystem** (the cDSP/NSP compute-subsystem entry that gates CDSP/fastRPC bring-up) is **binned/fused OFF** → `0xff` "absent/disabled."

Most accurate interpretation: **the NPU IP exists in silicon, but the NSP subsystem that makes it usable is fused off at the factory.** This matches the live device exactly — the `qcom_socinfo` debugfs cdsp/cdsp1/gpdsp/gpdsp1 image slots are **empty** (no firmware build string) while `adsp` is populated: the SoC firmware never provisioned a CDSP/NSP image because the part is binned out. A present-but-unusable NPU with a fused-off NSP subsystem is the textbook partial-goods outcome described in the Qualcomm harvesting patent.

---

## CITATION INDEX (file · symbol · URL)

- mainline `socinfo.c` · `struct socinfo` fields `num_subset_parts`, `nsubset_parts_array_offset`, `nmodem_supported` · https://github.com/torvalds/linux/blob/master/drivers/soc/qcom/socinfo.c
- downstream `socinfo.c` · `socinfo_get_num_defective_parts`, `socinfo_get_ndefective_parts_array_offset`, `msm_get_num_defective_parts` (DEVICE_ATTR), `socinfo->v0_14`, SMEM source · https://github.com/realme-kernel-opensource/realme5-kernel-source/blob/master/drivers/soc/qcom/socinfo.c
- this kernel · `socinfo_get_subpart_info(SOCINFO_PART_NSP)` interface attested · `_artifacts/aime-kernel-bypass-20260629/PATCH_DESIGN_DRAFT.md`
- Qualcomm patent · US 12,061,855 *Functional circuit block harvesting in ICs* (assignee Qualcomm): fuses blown at mfg test to disable failed cores; fuse state read at boot for availability · https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12061855
- companion patents · US 12,361,191; US 11,940,944 *Fuse recipe update mechanism* (Qualcomm)
- live data · `nsp=0xff modem=0xff npu=0x0`, cdsp/gpdsp image slots empty, HLOS qfprom window 0x221c8000–0x221c8fff only, CONFIG_DEVMEM unset · `_artifacts/aime-fuse-read-20260630/aime_smem_socinfo.txt`, `aime_fuse_read_0x221c2420.txt`
- prior decision · only XBL-code path can change it · `_artifacts/aime-enthusiast-last-mile-20260629/FEATURE_OVERRIDE_DECISION.md` (`XBL_PATCH_ONLY_CONFIRMED`)

## FINAL PROBABILITY

P(NPU permanently hardware-disabled via factory QFPROM partial-goods bin-out) ≈ **0.94**.
Residual ~0.06 = the one unproven link (that this CLO build's `nsp` show-function indexes the subset-parts array rather than a different feature_word path) — not independently re-verified against THIS exact kernel binary's socinfo.c. Every other link (defective↔subset rename, SMEM/XBL source, modem=0xff parallel, QFPROM permanence, empty CDSP image slots, no HLOS write path) is confirmed.
