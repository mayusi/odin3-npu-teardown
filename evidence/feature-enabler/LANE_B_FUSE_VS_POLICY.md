# Lane B — Fuse vs. Policy: Is NSP-disable hardware-permanent or firmware-changeable?

Generated: 2026-06-30 (OFFLINE analysis only — no device/adb/fastboot/EDL touched by this analysis)
Target: Odin 3 (device `sun`, SoC `CQ8725S` = Qualcomm SM8750 "sun" / Snapdragon 8 Elite), SKU `CQ8725S-3-AA`, soc_id 706, rev 2.0
Question: Is `nsp=0xff` / `ro.boot.vendor.qspa.nsp=disabled` a **soft policy value** (editable in principle) or a **blown QFPROM fuse** (physically permanent)?

---

## HEADLINE VERDICT (revised after live read-only socinfo + XBL rule decode)

**FUSE-DRIVEN, hardware-sourced — the NSP-disable value is read directly from a QFPROM fuse by XBL.** This is a material upgrade in certainty over the earlier "firmware-table / RetailImages" hypothesis, which is now **falsified** as the primary gate.

The decisive chain (all from existing artifacts):
1. XBL contains a feature-provisioning rule that does `value = read32(0x221c2420) & 0xff; feature_word[15] |= value`, and Linux exposes `feature_word[15]` as `/sys/devices/soc0/nsp`. The observed `nsp=0xff` **is the byte XBL read out of `0x221c2420`.**
2. `0x221c2420` is confirmed to lie inside the SM8450/SM8750-family **QFPROM ECC-corrected fuse region `0x221C2000–0x221C3fff`** (whole fuse space `0x221C0000–0x221Cbfff`). It is a **hardware fuse-shadow row**, not a config blob.
3. The live socinfo defective-parts bitmap shows `nsp=0xff` sharing the **exact same `0xff` "part-not-available" encoding as `modem=0xff`** — and `modem=0xff` is a *known-true permanent hardware bin-out* (this handheld has no cellular). Same table, same encoding, same QFPROM-subset-part source.

So the NSP subsystem is gated off by a **blown/programmed QFPROM fuse bit**, the same binning mechanism Qualcomm uses to disable the modem on this SKU.

**Nuance that keeps the project from being flatly "impossible":** `npu=0x0` proves the NPU compute IP is *present on the die* (see §2), and the fuse's *effect* is applied by an XBL software rule that is in principle NOP-able (mask `0xff`→`0x00`). But that XBL edit is MBNv7/SHA-384 **hash-covered** (no signing key) — so the fuse value cannot be suppressed with current resources. Net: **silicon likely capable, disable is fuse-sourced and signing-sealed → effectively permanent for this project unless a signing/authority path appears.**

---

## 1. The single strongest piece of evidence (FUSE, not policy)

`aime-npu-unlock-20260628/AIME_NPU_UNLOCK_CURRENT_STATE_20260628.md:47-67` — the decoded XBL rule:

```
Rule bytes:  20 24 1c 22 00 00 00 00 ff 00 00 00 00 00 0f 00
srcPtr = 0x221c2420   mask = 0xff   rshift=0  lshift=0   part = 15
value  = ((read32(0x221c2420) & 0xff) >> 0) << 0
feature_word[15] |= value
=> /sys/devices/soc0/nsp = 0xff
```

This is XBL **reading a hardware fuse address and copying it into the NSP feature slot**. The disable is not a string table — it is a fuse read.

Address identity confirmed against public Qualcomm QFPROM layout (LKML SM8450 QFPROM thread, https://lkml.iu.edu/2403.1/08346.html):
- Entire QFPROM fuse space: **`0x221C0000–0x221Cbfff`**
- **ECC-corrected fuse range: `0x221C2000–0x221C3fff`** ("intended for functional use of the QFPROM stored data by software")
- `0x221c2420` lands inside the corrected-fuse range → it is a **QFPROM corrected/ECC fuse-shadow row**.
Corroborated by `aime-npu-unlock-20260628/AIME_NPU_UNLOCK_OFFLINE_DECODE_RESULT_20260628.md:8,14,27` (verdict `QFPROM`, "likely corrected/fuse-shadow QFPROM space", confidence 0.75) and `..._CURRENT_STATE...:82,104-113` ("No NSP/CDSP fuse cell is visible to Linux… XBL is reading raw/private fuse space Android intentionally does not expose"). The Linux-visible HLOS node `qfprom@221c8000` is the controller's *corrected base*; the rule reads the lower private fuse aperture.

The earlier "policy" lead is explicitly **falsified**: `..._CURRENT_STATE...:77` — "`xbl_config RetailImages` — Falsified as primary fix. A working SM8750 donor also omits CDSP from `RetailImages`." Adding `FULL_CDSP` to RetailImages was a red herring; the true gate is the fuse-fed `feature_word[15]`.

---

## 2. Reconciling `npu=0x0` (present) vs `nsp=0xff` (absent)

Live bitmap (`aime-npu-feasibility-20260630/live_readonly_probe/LIVE_CAPABILITY_EVIDENCE.md:13-16`):
```
npu=0x0   nsp=0xff   modem=0xff   gpu/video/camera/audio/spss/wlan/display=0x0   nav=0x1
```

These are two different granularities in Qualcomm's `socinfo` subset-parts table (kernel enum `SOCINFO_PART_NSP` etc., confirmed at `aime-kernel-last-mile-20260629/ACTIVE_DT_CDSP_MAP.md:79-86` → `include/soc/qcom/socinfo.h:93`; populated by `socinfo_get_subpart_info()` from the SMEM subset-parts array, which boot firmware fills from QFPROM subset/defective-part fuses; `0x0`=available, `0xff`=not available/binned):

- **`npu=0x0`** → the Hexagon **NPU compute fabric / tensor IP is physically present** on the die. The silicon is there.
- **`nsp=0xff`** → the bootable **NSP *subsystem*** (the cDSP processor instance that loads `cdsp.mdt`, that CDSP-remoteproc/`q6v5_pas` binds, that exposes `/dev/fastrpc-cdsp`) is marked **not-available** in the subset-parts table.

So the IP block exists, but the **subsystem boot-enable for it is fuse-gated off.** This is precisely the profile of a *binned/partially-fused* part: the macro is on silicon, the per-subsystem enable fuse is programmed off. Supporting "silicon present but only gated": the full CDSP DT node exists with all rails/clocks (`status=no` only), and full CDSP kernel plumbing is wired (`smp2p-cdsp, qcom,cp_cdsp, cma-secure-cdsp, glink_pkt_data_cdsp, rdbg_cdsp` — `LIVE_CAPABILITY_EVIDENCE.md:20-38`).

**Direct answer to "is the Hexagon NSP silicon physically present & capable, with nsp=0xff being a policy gate":** *The NPU IP is almost certainly physically present (npu=0x0). But nsp=0xff is NOT merely a policy gate — it is a fuse-sourced subset-part disable of the NSP subsystem, encoded identically to the permanent modem bin-out.* So: **silicon present = likely YES; "just policy" = NO (it is fuse-driven).**

---

## 3. subset_parts = 9040 / num_subset_parts = 0x14 (20)

`num_subset_parts=0x14` (20 parts), `subset_parts=9040` is a summary/offset value from the SMEM array, not a directly human-decodable per-part membership list in the captured evidence. The authoritative per-part membership we *do* have is the decoded per-feature bitmap itself (`npu=0x0` present-set, `nsp=0xff`/`modem=0xff` absent-set). **The NSP falls in the ABSENT/binned-out set** by its `0xff` marker — that is the operative fact; the raw `9040` packed value does not override the explicit `nsp=0xff` field. (To fully decode `9040` into the 20-part membership array would require reading the `nsubset_parts_array` from SMEM with root — see §5 — but it is not needed to place NSP in the absent set, which `nsp=0xff` already does.)

---

## 4. The signing wall (why even a NOP can't be applied)

The fuse's *effect* is delivered by the XBL rule, and the minimal fix is to neuter the mask:
`..._CURRENT_STATE...:119-135` — `offset 0x10935c: ff -> 00` ⇒ `read32(0x221c2420) & 0x00 = 0`, NSP slot no longer receives `0xff`. Verifier: **`CANDIDATE_PRECISE_BUT_HASH_COVERED`**. The patched segment SHA-384 (range `0xd3574..0x10c574`, stored hash at `0x121864`) changes → fails XBL auth (`..._CURRENT_STATE...:145-157`). No SM8750 MBNv7 signing chain is available (`xbl-offline-certainty-20260628/FINAL_OFFLINE_GO_NO_GO_REPORT.md:43-51`). This is a **signing wall on top of a fuse**: even though the rule is logically patchable, it can't be re-signed.

---

## 5. The ONE read-only thing that settles "physically dead vs fuse-gated-but-IP-present"

The discriminating fact never empirically obtained is the **actual value at `0x221c2420`** read live, plus the SMEM subset-parts array decode:

1. **`peekdword 0x221c2420` via a firehose loader that advertises `peek`.** Every prior loader refused (`runtime_peek_confirmed:false`, `peek_skipped_reason:"Firehose nop did not advertise peek"` — `aime-npu-unlock-20260629/next_move_20260629/next_move_evidence.json:277-281`). The iQOO-13/bkerler `peekdword` path is the named candidate (`..._CURRENT_STATE...:181-187`). Reading the raw fuse word tells you whether the NSP bit is a single programmed disable bit (gate) vs the whole subsystem fused dead. **Read-only, but requires EDL.**
2. **Cheaper:** rooted decode of the SMEM `nsubset_parts_array` (`socinfo_get_subset_parts()` / `socinfo_get_subpart_info(SOCINFO_PART_NSP)`) to see whether NSP is reported "present-but-disabled-subpart" vs "absent." Needs root only, no EDL.

Either read converts the current strong inference into a hard fact. Note: **neither can change the outcome for the project** — a clear fuse bit still can't be acted on (XBL is hash-sealed), and a set fuse bit confirms permanence. The reads only resolve *why*, not *whether*.

---

## 6. Bottom line

| Question | Answer | Confidence | Basis |
| --- | --- | --- | --- |
| Fuse or policy/firmware-table? | **FUSE-DRIVEN** (QFPROM-sourced) | HIGH | XBL rule `read32(0x221c2420)&0xff → nsp slot`; `0x221c2420` ∈ QFPROM corrected range; `nsp=0xff` == `modem=0xff` bin-out encoding |
| Is the Hexagon NPU **silicon** physically present? | **Likely YES** | MEDIUM-HIGH | `npu=0x0` (NPU IP present); full CDSP DT node + kernel plumbing present (gated, not absent) |
| Is `nsp=0xff` "just a policy gate"? | **NO** — it is a fuse-sourced subset-part disable | HIGH | value is literally read from a QFPROM fuse row by XBL; not from RetailImages (that path falsified) |
| Is the project effectively impossible? | **YES with current resources** (not strictly proven "silicon-dead") | HIGH (practical) | fuse-fed disable + XBL hash-seal; no signing key; live peek of the fuse never obtained |

**Practical:** Earlier optimism rested on RetailImages being the switch — that is now falsified. The real gate is a **QFPROM fuse read** wired into XBL's feature-provisioning, sharing the modem's permanent bin-out encoding. The NPU compute IP is probably on-die (`npu=0x0`), so the part is more "subsystem fused-off / binned" than "macro absent" — but for enablement purposes that distinction is academic: the only firmware switch (NOP the XBL mask) is hash-sealed and unsignable, and the fuse itself is QFPROM-permanent. **This moves the verdict from "alive, blocked on signing" toward "fuse-gated and signing-sealed = dead-end for a software/firmware unlock."** The single remaining empirical unknown (live `peekdword 0x221c2420`) would explain the mechanism but cannot revive the path.
