# Frequently Asked Questions

Honest answers to the questions that come up every time someone reads about a hardware-fused NPU. No clickbait, no "maybe someday" hedging where the evidence doesn't support it.

---

## "Can't AYN just push a software update to enable the NPU?"

**No.** This is the most important misconception to clear up.

AYN does not control the state of the NSP fuse. The fuse is a **one-time-programmable (OTP)** register inside the Qualcomm QFPROM array. It is blown — physically — during manufacturing test at Qualcomm's factory, before the SoC is ever packaged, sold, or handed to AYN. AYN receives a chip that already has the NSP subsystem marked absent at the silicon level.

A software update operates at the Android/firmware layer, above the bootloader, above XBL, and many layers above the fuse. The fuse is a hardware bit that XBL reads once at power-on and acts on. No Android OTA, no kernel update, no FeatureEnabler license pack, no signed firmware bundle can reach below XBL to write to an OTP register. OTP means one-time-programmable — you can write it exactly once, in one direction, and it is permanent.

Even if AYN wanted to "re-enable" the NPU, Qualcomm would need to supply a different binning of the silicon. AYN cannot fix this with software, because the decision was made before they touched the chip.

---

## "Is it REALLY permanent? How sure are you?"

**~94% confident it is permanent.** Here is what that number means and where it comes from.

Three independent lines of evidence all point to the same cause — a factory-blown QFPROM OTP fuse:

1. **Live on-device socinfo read** (`uid=0`, read-only): `/sys/devices/soc0/nsp` = `0xff`. In the Qualcomm `subset_parts` table, `0xff` is the sentinel value meaning "this component was binned out." The same `0xff` appears for `/sys/devices/soc0/modem`, which is a known-absent component on the CQ8725S IoT SKU — validating that the encoding means what we think it means. This table is written by XBL at boot from QFPROM data.

2. **FeatureEnabler trustlet disassembly**: The SM8750 `featenabler.mbn` binary contains a call to `qsee_is_sw_fuse_blown(22)`, where index 22 maps to `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]`. This is a hardware OTP query, not an RPMB soft fuse. AYN's factory FeatureEnabler has no NSP license entry. The trustlet cannot enable NSP because the gate it checks is hardware.

3. **Kernel source + Qualcomm patent US 12,061,855**: The CLO `socinfo.c` `subset_parts` array is documented in the patent as `defective_parts` — components physically absent due to manufacturing binning. The NSP entry is populated by XBL from QFPROM at boot. The patent explicitly confirms this is a factory OTP process.

The residual **~6%** reflects a single physically unverifiable link: the raw fuse register `0x221c2420` is outside the HLOS-mapped QFPROM window (`0x221c8000–0x221c8fff`), so it cannot be read directly from Android. What we read (`nsp=0xff`) is XBL's decoded output — which is authoritative — but we cannot independently inspect the raw bit. All evidence is consistent with a blown OTP; none of it is consistent with any alternative explanation.

In practical terms: there is no known path that would change this outcome, and no evidence that suggests one exists.

---

## "The /sys/devices/soc0/npu node shows npu=0x0 — doesn't that mean the NPU is present?"

**The NSP subsystem that makes it usable is fused off — that is the operative, well-evidenced fact. Whether the underlying compute IP is literally present on the die is the most likely reading, but is an inference.**

`npu=0x0` (a separate socinfo field from `nsp`) most likely indicates the NPU/AI-accelerator block is not flagged as binned out. However, the exact per-SoC meaning of the `npu` socinfo field is not documented in public kernel source — the SM8750 `subset_parts` part-name→ID mapping is proprietary CLO firmware and is not present in upstream `socinfo.c`. So "the NPU compute hardware is etched into the die" is the most plausible reading of `0x0`, not a proven fact.

What is firmly established: `npu=0x0` uses a different encoding than the `0xff` binned-out sentinel. The `nsp=0xff` value is the operative one — it tells XBL (and every subsequent layer) that the NSP subsystem does not exist for this device. Regardless of what compute logic may be present in silicon, CDSP is never brought up, FastRPC domain 3 is never registered, and `/dev/fastrpc-cdsp` is never created. The root cause is `nsp=0xff`.

Think of it like this: CPU cores can be fused off in binned chips even if the physical gates are etched into the die. What matters is whether the system-level activation exists — and for the NSP, it definitively does not.

---

## "Could a custom kernel enable it?"

**No.** The kernel cannot reach the fuse, and the CDSP is disabled before the kernel ever loads.

The sequence at boot is:

1. XBL (eXtensible Bootloader) runs — it has full QFPROM aperture access and reads `0x221c2420`.
2. XBL writes the decoded results to SMEM and sets the device-tree CDSP node to `status = "disabled"`.
3. ABL and the Android bootloader chain run.
4. The Linux kernel loads — it reads the device tree that XBL already prepared.
5. The kernel's remoteproc / `qcom_q6v5_pas` driver sees CDSP `status = "disabled"` and skips bring-up entirely.

A custom kernel would need to run *before* XBL to change this outcome — which means it would need to be signed with Qualcomm's SecBoot key (SHA-384), because XBL is part of the trusted boot chain. That key is not publicly available.

Even if the kernel tried to set the CDSP DT node to `status = "okay"` at runtime, the CDSP power domain, clocks, and PAS domain would still not be available — they were never initialized because XBL never set them up. The hardware bring-up that enables CDSP happens in XBL, before Linux exists.

---

## "What about rooting / Magisk — couldn't a root method reach the fuse?"

**Root was obtained and used. It cannot reach the fuse.**

This investigation was conducted with full root (`uid=0` via PServer Binder, `context=u:r:pservice:s0`, `boot_completed=1`). Root access gave unrestricted access to every path that HLOS can reach. The results:

- `/sys/devices/soc0/nsp` = `0xff` — readable, confirms the verdict.
- `/proc/iomem` QFPROM window: `0x221c8000–0x221c8fff` — the window available to HLOS does **not** cover `0x221c2420`. Root cannot expand the QFPROM aperture.
- `/dev/mem`: not present — `CONFIG_DEVMEM` is compiled out of the kernel. No physical memory mapping.
- QFPROM nvmem sysfs entries: cover only the HLOS-permitted range.

Root means "Linux superuser." The fuse is not a Linux resource — it is a hardware register that XBL reads before Linux exists. Root cannot write OTP fuses. Even if it could reach the fuse address, OTP means the bit is physically unchangeable once set: you cannot blow a fuse "backwards."

Magisk, KernelSU, APatch, and any other root method operate at or below the Android framework but above XBL. None of them can alter the hardware state that XBL read before they ran.

---

## "Could EDL / Firehose flash fix it?"

**No on two counts: fuses are not writable via Firehose, and flashing modified partitions bricks the device.**

**On the fuse:** EDL (USB PID `0x05c6:0x9008`) was successfully entered, and the iQOO 13 SM8750 Firehose programmer was loaded. The programmer supports read-only storage operations and partition backup. It does not support `peek` (direct memory read) or `poke` (direct memory write) — the loader responds with `NAK` to any peek request. Even if `peek` were supported, QFPROM fuses are write-once hardware registers. Poking a value into a fuse that is already blown has no effect; the hardware is permanently set.

**On flashing modified partitions:** This was tested directly, and the device bricked. Modifying `vendor_dlkm` changed its `dm-verity` root digest from `d0f1abeb...` to `d6572a40...`. The `vbmeta_a` partition is **signed** and contains the authoritative verity descriptor. The mismatch caused `veritymode=eio` on next boot — every read from the modified partition returned I/O error, and the device became fully unresponsive. Full stock `super` partition restore via EDL was required to recover.

The signed `vbmeta` chain means: any modification to any verified partition without a matching Qualcomm-signed `vbmeta` update will brick the device. AYN and Qualcomm hold the signing keys. Community members do not.

---

## "Will a future OTA from AYN help?"

**No.** An OTA cannot reverse a blown fuse.

An Android OTA updates software: Android system partitions, kernel, vendor blobs, firmware images. It cannot write to QFPROM OTP registers. The NSP fuse was blown before AYN received the SoC; AYN cannot ship an OTA that "unblows" it, because AYN never had access to the fuse in the first place.

If Qualcomm were to issue a "FeatureEnabler token" that unlocked NSP in software, that would bypass the OTP fuse entirely — but the FeatureEnabler trustlet code shows it calls `qsee_is_sw_fuse_blown(22)`, a hardware OTP query. There is no software fast-path around it. The trustlet itself is designed to prevent soft-fuse workarounds.

Qualcomm has never issued a FeatureEnabler token for a hardware-OTP-disabled feature on a shipping device, and there is no mechanism in the existing trustlet to accept one.

---

## "What WOULD it take to run NPU workloads on this hardware?"

**A different, un-binned device.**

The only path to NPU compute on SM8750 silicon is a device where Qualcomm did not blow the NSP fuse — a device that shipped with a full SM8750 SKU (like the OnePlus 13, Xiaomi 15, or iQOO 13). Every one of those devices runs CDSP normally, QNN HTP works out of the box, and `nsp` is not `0xff`.

The AYN Odin 3 received CQ8725S silicon — the Dragonwing industrial/IoT binning of SM8750. That binning exists because Qualcomm tests each die and disables subsystems that fail their test or that are intentionally gated for market segmentation. The CQ8725S spec does not include NSP. That decision is final.

If AYN releases a future Odin product on a different SoC revision or SKU that includes NSP, that would be a different device. Retrofitting the Odin 3 is not possible.

---

## Quick Reference

| Question | Short Answer |
|----------|-------------|
| Can AYN push a software fix? | No — they don't control the fuse |
| Is it really permanent? | ~94% yes — three independent proofs; 6% = one unreadable register |
| The npu node shows 0 — present? | IP present, NSP subsystem (the usable part) is fused off |
| Custom kernel? | No — CDSP is disabled by XBL before the kernel loads |
| Root / Magisk? | Root was used for this investigation; it cannot reach below XBL |
| EDL / Firehose? | Fuses aren't writable; modified partitions brick the device (confirmed) |
| Future OTA? | OTAs cannot unburn OTP hardware fuses |
| What would work? | Different un-binned silicon — a different device |
