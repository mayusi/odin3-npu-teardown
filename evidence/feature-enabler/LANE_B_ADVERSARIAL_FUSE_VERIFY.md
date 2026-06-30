# Lane B — Adversarial Verification of the "Permanent Blown QFPROM Fuse" Claim

Generated: 2026-06-30
Mode: Adversarial (try to REFUTE first), offline reasoning + web research. Read-only. No device touched.

---

## The claim under test

> On the AYN Odin 3 (SM8750 "sun", SoC model CQ8725S-3-AA), the NPU/NSP is disabled because XBL reads `0x221c2420`, masks `0xff`, publishes it as socinfo subset-part #15 (nsp)=0xff. Because `0x221c2420` is inside the QFPROM ECC-corrected read range (`0x221c2000-0x221c3fff`), the disable is a **BLOWN ONE-TIME QFPROM FUSE and therefore PHYSICALLY PERMANENT** — no firmware/software unlock possible. Rated "FUSE — hardware-permanent, project impossible, high confidence."

## VERDICT: **UNRESOLVABLE-WITHOUT-PEEK** — weight leans *substantially* toward permanent factory part-binning, but "high-confidence blown fuse" is **NOT proven** and the stated reasoning contains a genuine logical defect.

The honest grade is: **likely-permanent (≈0.7), but the specific "high confidence" rating is unearned.** The single load-bearing inference in the strong claim — *"corrected range ⟹ blown OTP ⟹ permanent"* — is a **non-sequitur** and is independently contradicted by the Qualcomm kernel maintainer. The conclusion may still be right, but for a *different* reason (socinfo subset-parts = factory partial-goods binning), and that reason is itself not yet proven for part #15 specifically without a live read.

---

## Why the strong claim's REASONING is defective (the refutation that actually lands)

**"Corrected range" means ECC-corrected, NOT writability-locked, NOT permanent.** This is the core flaw. Qualcomm kernel maintainer Mukesh Ojha, on the SA8775P QFPROM thread for this exact `0x221c....` block family, states:

> "ECC corrected range is this 0x221C2000-0x221C3fff and High level OS does have a access to ECC range however, they are not recommended for SW usage."
> — https://lkml.iu.edu/2405.3/05957.html

Two things follow that damage the strong claim:
1. **"Corrected" is purely about ECC error-correction on the read path**, distinguishing it from the "raw" aperture (`0x221c0000-0x221c1fff`). It says nothing about whether the underlying cell is a blown OTP bit, a secure-shadow register, or a provisioned config row. The address being inside `0x221c2000-0x221c3fff` is therefore **zero evidence for permanence**. The project's own primary attribution file already conceded exactly this: it graded the finding `QFPROM_CORRECTED_RANGE_POLICY_INPUT_FOUND`, explicitly "still WEAKER than FUSE_POLICY_LOCKED," and explicitly "we still have NOT proven whether the value at boot is permanent OTP, a secure shadow, an OEM policy field, or a masked boot-stage value." (`ADDRESS_ATTRIBUTION_0x221c2420_20260628.md`.) The "high-confidence FUSE" rating **upgraded** that primary evidence without new facts — that's the inflation this lane is meant to catch.
2. The maintainer says **HLOS *does* have access to the ECC range**. The Odin simply doesn't *map* `0x221c2420` (its DT only exposes `qfprom@221c8000`, size `0x1000`). So even the "Android can't read it" framing is a board-DT choice, not a hardware lock — mildly weakening the "untouchable fuse" picture.

So: the address argument is a category error (ECC ≠ OTP). The verdict cannot rest on it.

---

## (a) What `subset_parts` is actually sourced from (per Qualcomm/Linux evidence)

This is the part that makes the conclusion *probably still true*, just for the right reason.

- Modern Qualcomm SoCs (SM8550+) added **Product Code (pcode) + Feature Code (fcode)** to socinfo "for precisely identifying the specific SKU and the precise speed bin" — Konrad Dybcio, kernel socinfo work. https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg488817.html and follow-ups. The `socinfo_params` struct carries `num_subset_parts` + `nsubset_parts_array_offset` (introduced ~SOCINFO_VERSION 0,14), plus `feature_code`/`pcode`/`nproduct_id`. Confirmed in mainline: https://github.com/torvalds/linux/blob/master/drivers/soc/qcom/socinfo.c and rendered https://codebrowser.dev/linux/linux/drivers/soc/qcom/socinfo.c.html
- **Provenance: SMEM, populated by boot firmware.** socinfo.c reads `qcom_smem_get(QCOM_SMEM_HOST_ANY, SMEM_HW_SW_BUILD_ID, ...)`. The kernel does **not** read the fuses itself; PBL/XBL assemble the socinfo table (including the per-part "subset"/defective-parts array and feature/product codes) and place it in SMEM. On Qualcomm, the authoritative source for "which SKU / which blocks / which speed bin" is the **QFPROM feature/config fuses read at boot** — this is the standard part-identification fuse layer, the same family the Qualcomm Linux Security Guide calls feature/config fuse rows (https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/appendix-fuse-configurations.html).

**Honest caveat:** mainline socinfo.c carries **no comment** literally labeling `subset_parts` as "defective/disabled blocks," and Qualcomm's public docs do not publish the per-bit feature-word map. So "subset-part #15 = NSP, sourced from the factory partial-goods fuse table" is a **strong, well-supported inference** (XBL rule literally maps part 15 → socinfo nsp on two independent devices), but it is **not a verbatim-documented fact.** The semantic chain is solid; the final fuse-bit attribution is inferential.

## (b) Is CQ8725S a known NPU-disabled IoT SKU?

**No — and this is the strongest single point AGAINST "NPU permanently fused off as a SKU decision."**
- CQ8725S is confirmed an **embedded/IoT binned variant** of CQ8750S (the Dragonwing **Q-8750** AIoT part), "practically identical" to the Snapdragon 8 Elite, with the documented removals being the **Cellular Modem-RF system and camera/ISP processing** — **not** the NPU. https://retrohandhelds.gg/your-odin-3-might-not-actually-use-the-snapdragon-8-elite-but-theres-more-to-the-story/
- The Dragonwing **Q-8750 ships the Hexagon V79 NPU at 77 dense TOPS as a headline feature** — the AI engine is the *whole point* of the AIoT SKU, not something binned away. https://www.cnx-software.com/2026/01/06/qualcomm-dragonwing-q-7790-and-q-8750-aiot-socs-target-ai-enhanced-drones-cameras-tvs-and-media-hubs/

So there is **no public SKU in which this silicon ships with the NSP deliberately disabled.** That means the Odin's `nsp=0xff` is **either** (i) a genuine per-die partial-goods bin (this specific die's Hexagon failed test and was fused out, while CPU/GPU passed), **or** (ii) an AYN/board-level config/provisioning state — **not** a known catalog SKU choice. Option (i) is permanent; option (ii) may not be. The catalog evidence does not let us pick.

## (c) Single strongest fact for the verdict (with URL)

For *"weight leans permanent"*: Qualcomm demonstrably ships **per-die partial-goods binning of functional blocks on this exact silicon** — **SM8750-3-AB** is a catalogued Snapdragon 8 Elite bin **with one middle CPU core disabled**. https://www.phonearena.com/news/galaxy-z-fold-7-might-use-binned-version-snapdragon-8-elite_id166773 . Block-level disable-by-binning on SM8750 is real and routine; `nsp=0xff` being the same kind of factory disable is entirely plausible and, if so, is permanent. (Reinforced by the live `modem=0xff` reading from the *same* subset-parts table — see (d).)

For *"but not proven / maybe provisionable"*: Qualcomm publicly documents **"SoftSKU feature packs"** that are **installed/upgraded in software** on a deployed SoC, and **devcfg enabled from QTEE** — i.e., not every per-feature gate on Qualcomm silicon is a permanent fuse; some are signed, software-licensable feature state. https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/upgrade-qualcomm-wes-feature-pack.html and https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-11/enable-device-devcfg-from-qtee.html . This is the concrete existence-proof that "lives in/near the fuse/config layer" ≠ "permanently blown."

## (d) Does the modem=0xff analogy strengthen or weaken "permanent"?

**It strengthens "permanent" — and is the best corroborator in the whole dossier.** Live Odin reads `modem=0xff` from the **same socinfo subset-parts table** that yields `nsp=0xff`. The CQ8725S **documentably has the modem removed** as part of its IoT binning (Retro Handhelds, above). So we have a *known-true* partial-goods/binned-out block (modem) producing `0xff` via the identical table and code path. That establishes, on this device, that `0xff` in this table **is** how a genuinely binned-out subsystem is encoded — which makes `nsp=0xff` looking like the same factory disable the more likely reading.

**The honest counter:** "modem removed" on an IoT part can also be a packaging/RF-not-bonded decision rather than a die-level Hexagon-style fuse, and the modem case tells us nothing about whether the *NSP* disable is a hard fuse vs a config field — it only shows `0xff` *can* mean "binned out." It raises the prior for permanence; it does not prove it for part #15.

---

## Net assessment (adversarial bottom line)

| Vector | Direction |
|---|---|
| "Corrected range ⟹ permanent OTP" | **FALSE reasoning** (ECC ≠ OTP; maintainer-contradicted). Strong claim's stated logic fails. |
| XBL rule part 15→nsp, mask 0xff | Real, cross-device (Odin + Realme) — but it's **common Qualcomm boilerplate**, source value still unread. |
| socinfo subset_parts = factory partial-goods, fuse-sourced at boot | **Well-supported inference**, not verbatim-documented for bit 15. → leans permanent |
| CQ8725S a catalogued NPU-disabled SKU? | **No** — NPU is a headline feature of this silicon. → opens non-SKU / possibly-config door |
| SM8750 does per-die block binning (8-3-AB core-disable) | **Yes, documented.** → leans permanent |
| SoftSKU feature packs / QTEE devcfg exist | **Yes, documented.** → some Qualcomm feature gates are software-provisionable, not fuses |
| modem=0xff via same table, modem really binned out | **Yes.** → leans permanent (strongest corroborator) |

**Cannot be closed without one read.** Everything hinges on whether `0x221c2420` is a blown OTP cell vs a (re)provisionable config/shadow row, and that is *exactly* the byte no current loader can `peek`. The weight of evidence (block-level binning is real on SM8750; modem=0xff via the same table is a genuine binned-out block; subset_parts is the fuse-derived partial-goods layer) leans toward **permanent factory part-binning, ~0.7**. But the existence of SoftSKU/QTEE feature provisioning and the absence of any catalogued NPU-disabled SKU keep a **real (~0.3) door open** that this is a locked-but-provisionable feature/config value, not a dead silicon block.

## (e) Bottom line for the user (one sentence)

The NPU is **most likely permanently dead by factory part-binning (~70%), but this is NOT proven and the "high-confidence blown-fuse" rating is overstated — its logic (corrected-range = permanent) is wrong, and a ~30% chance remains that `nsp=0xff` is a locked-but-provisionable config/feature value rather than a blown OTP bit; the question is genuinely unresolvable without a single read-only peek of `0x221c2420`.**

---

## Sources
- ADDRESS_ATTRIBUTION_0x221c2420_20260628.md (project primary evidence — graded weaker-than-FUSE)
- SA8775P QFPROM corrected/raw ranges, maintainer Mukesh Ojha: https://lkml.iu.edu/2405.3/05957.html
- Linux socinfo.c (mainline): https://github.com/torvalds/linux/blob/master/drivers/soc/qcom/socinfo.c — rendered: https://codebrowser.dev/linux/linux/drivers/soc/qcom/socinfo.c.html
- pcode/fcode mechanism (Dybcio): https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg488817.html
- CQ8725S binning (modem+camera removed, not NPU): https://retrohandhelds.gg/your-odin-3-might-not-actually-use-the-snapdragon-8-elite-but-theres-more-to-the-story/
- Dragonwing Q-8750 ships Hexagon V79 NPU @77 TOPS as headline: https://www.cnx-software.com/2026/01/06/qualcomm-dragonwing-q-7790-and-q-8750-aiot-socs-target-ai-enhanced-drones-cameras-tvs-and-media-hubs/
- SM8750-3-AB binned with a CPU core disabled (block-level binning on this silicon): https://www.phonearena.com/news/galaxy-z-fold-7-might-use-binned-version-snapdragon-8-elite_id166773
- Qualcomm QFPROM fuses (OTP, permanent once blown): https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/appendix-fuse-configurations.html
- Qualcomm SoftSKU feature packs (software-installed/upgraded features): https://docs.qualcomm.com/bundle/publicresource/topics/80-70018-11/upgrade-qualcomm-wes-feature-pack.html
- Qualcomm devcfg enabled from QTEE: https://docs.qualcomm.com/bundle/publicresource/topics/80-70020-11/enable-device-devcfg-from-qtee.html
