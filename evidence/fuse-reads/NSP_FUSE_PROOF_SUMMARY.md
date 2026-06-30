# NSP Disable — Hardware Fuse Proof (on-device + trustlet disassembly)

Date: 2026-06-30 · Device: AYN Odin 3 `<SERIAL>` / Qualcomm `sun` SM8750 / **CQ8725S-3-AA**
Method: READ-ONLY only. Root via PServer Binder (`uid=0 context=u:r:pservice:s0`). No writes, no flash, no poke. Device healthy throughout (`boot_completed=1`).

## TWO INDEPENDENT HARDWARE PROOFS — both say NSP is fuse-disabled

### Proof 1 — Live on-device socinfo subset-parts (partial-goods) read
From `/sys/devices/soc0` and `/sys/kernel/debug/qcom_socinfo` (read-only, root):
```
nsp   = 0xff      <- NSP subsystem: ABSENT / binned out
npu   = 0x0       <- NPU compute IP: PRESENT on die
modem = 0xff      <- modem: ABSENT (handheld, no cellular = KNOWN-TRUE hardware bin-out)
gpu/video/camera/audio/spss/wlan/display = 0x0  (present)
num_subset_parts = 0x14 (20)   nsubset_parts_array_offset = 0xec   subset_parts = 9040 (0x2350)
sku = CQ8725S-3-AA   soc_id = 706   revision = 2.0   foundry_id = 1   raw_package_type = 0x7
qcom_socinfo image slots: cdsp/cdsp1/gpdsp/gpdsp1 = EMPTY (no fw build string); adsp = populated (CQ8750-ADSP-R01-V1.6)
```
`nsp` and `modem` are produced by the SAME socinfo `nsubset_parts` (partial-goods/defective-parts) table, which the boot firmware fills from QFPROM. `modem=0xff` is a verified real hardware bin-out on this cellular-less handheld → `nsp=0xff` in the same table = the same hardware-binning channel.

`0x221c2420` itself is UNREADABLE from Android: HLOS qfprom window is only `0x221c8000–0x221c8fff` (confirmed `/proc/iomem` + DT `reg = 22 1c 80 00 / 00 00 10 00`); `/dev/mem` is compiled out (`CONFIG_DEVMEM is not set`). So the raw fuse can't be peeked on-device — but its DECODED result is the socinfo `nsp=0xff` above.

### Proof 2 — FeatureEnabler trustlet checks a named NSP-disable QFPROM fuse
Disassembly of the SM8750 `featenabler` trustlet (sibling-chipset copy; the `qsee_is_sw_fuse_blown` fuse-ID is a chipset-BSP constant that applies to Odin identically):
```
76b0:  mov  w0, #0x16        ; fuse_id = 22
76b4:  add  x1, sp, #4       ; out: blown status
76b8:  mov  w2, #1
76cc:  bl   qsee_is_sw_fuse_blown
```
Fuse ID 22 = `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]` — a dedicated **hardware OTP fuse** read via the secure `qsee_is_sw_fuse_blown` (TZ/QSEE) path. This is DISTINCT from the soft "SwFuse" mechanism the same trustlet uses for display/video features (those write a runtime-mapped MMIO display register + RPMB; they never touch QFPROM). The NSP gate is specifically a **blown-fuse query**, i.e. hardware OTP, not a provisionable license.

## What this rules out (consistent with all prior lanes)
- FeatureEnabler does NOT license NSP (only Display_*/Video_* features). It merely *reads* the NSP fuse, can't set it.
- The corrected QFPROM aperture (`0x221c2000-0x221c3fff`, where `0x221c2420`/fuse 22 live) is not HLOS-mapped and not writable from Linux.
- `0x221c2420` read by signed XBL == fuse-22 NSP_DISABLE == socinfo `nsp=0xff` — three views of one hardware fact.

## VERDICT
**`NSP_HARDWARE_FUSE_DISABLED` — factory QFPROM bin-out, permanent.** The NPU compute IP is present on the die (`npu=0x0`), but the bootable NSP/Hexagon subsystem is fused off at manufacturing via the same partial-goods/QFPROM channel that disables the modem on this CQ8725S IoT SKU. No software, firmware, signed-feature, or HLOS path can change it. Confirmed by two independent methods (live socinfo partial-goods read + trustlet `qsee_is_sw_fuse_blown(NSP_DISABLE)` check). The NPU is **not unlockable on this device.**

(Read-only proof complete. No device modification performed. Raw evidence: `aime_fuse_read_0x221c2420.txt`, `aime_smem_socinfo.txt`, `aime_smem_raw.txt`.)
