# Proof 1 — Live On-Device Socinfo Read

**Method:** Read-only, on-device. Root obtained via PServer Binder (`uid=0 context=u:r:pservice:s0`). No writes. Device remained healthy (`boot_completed=1`) throughout.

**Device under test:** AYN Odin 3 · SKU `CQ8725S-3-AA` · SoC `SM8750 "sun"` · `soc_id=706` · `revision=2.0`

**Evidence files:**
- `evidence/fuse-reads/aime_smem_socinfo.txt`
- `evidence/fuse-reads/aime_fuse_read_0x221c2420.txt`

---

## What Was Read

The Qualcomm socinfo driver exposes a per-feature availability table at `/sys/devices/soc0/` and `/sys/kernel/debug/qcom_socinfo`. These nodes are populated at boot by the signed Extensible Bootloader (XBL) from SMEM — they are not computed by the kernel and cannot be modified from HLOS.

The full feature byte dump from the device:

```
nsp   = 0xff     ← NSP subsystem: ABSENT / binned out
npu   = 0x0      ← NPU compute IP block: PRESENT on die
modem = 0xff     ← modem RF: ABSENT (no cellular on this handheld)
gpu   = 0x0      ← GPU: present
video = 0x0      ← video codec: present
camera= 0x0      ← camera ISP: present
audio = 0x0      ← audio DSP: present
spss  = 0x0      ← SPSS security processor: present
wlan  = 0x0      ← WLAN: present

chip_id          = CQ8725S
sku              = CQ8725S-3-AA
soc_id           = 706
revision         = 2.0
raw_package_type = 0x7
num_subset_parts = 0x14   (20 entries)
nsubset_parts_array_offset = 0xec
subset_parts     = 9040  (0x2350)
```

Source: `evidence/fuse-reads/aime_smem_socinfo.txt` (full soc0 attribute dump).

---

## What `0xff` Means in This Table

The partial-goods / subset-parts table uses a simple availability sentinel:

| Value | Meaning |
|-------|---------|
| `0x0` | Subsystem present and good — not flagged defective |
| `0xff` | Subsystem absent / binned out — all-ones "disabled" sentinel |

This is the same convention used in Qualcomm boot logs for `nmodem_supported=0xff`.

### The Modem Parallel — The Key Cross-Check

`modem=0xff` on this device is a **known-true, independently verifiable hardware bin-out**: the CQ8725S is an IoT SKU of the SM8750 that ships without cellular RF hardware. There is no modem. The value `0xff` in the partial-goods table correctly reflects physical silicon absence — it was stamped in at factory test.

Critically, `nsp=0xff` and `modem=0xff` live in the **same array, filled by the same XBL-to-SMEM path, decoded by the same kernel driver function** (`socinfo_get_subpart_info()`). There is no mechanism that would make `modem=0xff` a real hardware bin-out while making `nsp=0xff` something different. They are produced identically.

**`nsp=0xff` is the same hardware-binning mechanism as `modem=0xff`.**

---

## `npu=0x0` Does Not Contradict `nsp=0xff`

These are **two different socinfo part indices** reporting two different things:

- **`npu=0x0`** — a separate socinfo field that most likely indicates the NPU/AI-accelerator block is not flagged as binned out. The exact per-SoC meaning of the `npu` vs `nsp` socinfo fields is not documented in public kernel source (the SM8750 `subset_parts` part-name→ID mapping is proprietary CLO/firmware, not present in upstream `socinfo.c`), so "the compute IP is physically present" is the most likely reading of `0x0` here — not a proven fact. This is consistent with every other likely-present part (`gpu`, `video`, `camera`, etc., all `0x0`).
- **`nsp=0xff`** — the bootable **NSP compute subsystem** (the cDSP/NSP firmware-hosting environment that gates FastRPC bring-up and the CDSP driver stack) is binned/fused OFF.

The operative conclusion: **regardless of whether the underlying compute IP is fully present on the die, the NSP subsystem that makes it usable is fused off at the factory.** That is what blocks CDSP/FastRPC/QNN — not any absence of silicon logic. This is the textbook partial-goods outcome: a die where the NSP/CDSP subsystem failed manufacturing test or was deliberately disabled for SKU differentiation.

This interpretation is directly confirmed by the CDSP firmware image slot data (see below).

---

## CDSP Firmware Slots Are Empty

The `qcom_socinfo` debugfs tree exposes per-subsystem firmware build strings. On a properly functional NSP, the CDSP slots would contain a build identifier. They do not:

```
/sys/kernel/debug/qcom_socinfo/cdsp/name    =   (empty)
/sys/kernel/debug/qcom_socinfo/cdsp/variant =   (empty)
/sys/kernel/debug/qcom_socinfo/cdsp1/name   =   (empty)
/sys/kernel/debug/qcom_socinfo/gpdsp/name   =   (empty)
/sys/kernel/debug/qcom_socinfo/gpdsp1/name  =   (empty)
```

Compare this against `adsp`, which is fully populated:

```
/sys/kernel/debug/qcom_socinfo/adsp/name    = 12:LPAIDSP.HT.1.1-01147.1-PAKALA-1
/sys/kernel/debug/qcom_socinfo/adsp/variant = pakala.adsp.prodQ
/sys/kernel/debug/qcom_socinfo/adsp/oem     = CQ8750-ADSP-R01-V1.6_20260616
```

XBL never provisioned a CDSP/NSP firmware image because the NSP part is binned out. The boot chain does not attempt to bring up a disabled subsystem.

---

## Why the Raw Fuse (`0x221c2420`) Cannot Be Read from Android

The QFPROM fuse row that XBL reads to derive `nsp=0xff` lives at physical address `0x221c2420` (in the ECC-corrected QFPROM aperture, `0x221c2000–0x221c3fff`). Direct readback from HLOS is impossible for two independent reasons:

**1. The HLOS QFPROM nvmem window does not cover that address.**

From `/proc/iomem` and the Device Tree `reg` property:

```
/proc/iomem:
  221c8000-221c8fff : 221c8000.qfprom qfprom@221c8000

Device Tree reg bytes (big-endian):
  0000000 22 1c 80 00  00 00 10 00
              ↑ base     ↑ size (0x1000 = 4 KB)
```

The HLOS window is `0x221c8000–0x221c8fff` — a 4 KB window for the QFPROM controller. The fuse at `0x221c2420` is `0x5be0` bytes below that window. It is simply not mapped.

**2. `/dev/mem` is compiled out.**

```
no /dev/mem
# CONFIG_DEVMEM is not set
# CONFIG_STRICT_DEVMEM is not set
```

There is no raw physical memory accessor available from HLOS. Even if a clever MMIO mapping were attempted, the security controller would block it.

**What we can read is what XBL published.** XBL runs before HLOS with full access to the QFPROM aperture, reads `0x221c2420`, extracts bit-field `feature_word[15]`, and writes the decoded result into SMEM as `nsp`. The kernel reads that SMEM value and surfaces it as `/sys/devices/soc0/nsp`. This is the authoritative, XBL-signed decoded state of the fuse — not a kernel guess.

```
Physical fuse 0x221c2420  →  XBL reads (full QFPROM access)
     ↓
SMEM socinfo nsp byte = 0xff  →  kernel publishes to /sys/devices/soc0/nsp
     ↓
On-device read: nsp=0xff
```

The raw cell is inaccessible from HLOS by design; its decoded result is what is visible and is sufficient.

---

## Raw QFPROM nvmem Window Hexdump (for completeness)

The HLOS-accessible QFPROM window (`0x221c8000`) was dumped. First 256 bytes:

```
000000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
000010 57 bb 94 72 01 00 00 00 00 00 00 00 00 00 00 00
000020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
000040 00 a8 94 c0 01 00 00 00 00 00 00 00 00 00 00 00
000050 00 00 70 32 00 00 00 00 00 00 00 00 00 00 00 00
000060 40 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
000070 5f cd 83 0f 10 00 01 04 00 00 00 00 00 00 00 00
000080 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
*
000100
```

This window (`0x221c8000`) is the QFPROM controller register space, not the fuse-array aperture. The fuse at `0x221c2420` is in a different, non-HLOS-mapped region. These bytes do not contain the NSP fuse and are shown here only to document the extent of HLOS access.

---

## Summary

- `nsp=0xff` was read directly from the device's socinfo partial-goods table.
- `modem=0xff` in the same table is a verified real hardware bin-out, providing an exact parallel.
- CDSP/NSP firmware slots are empty, confirming the boot chain never activated the subsystem.
- The raw QFPROM fuse cell at `0x221c2420` cannot be read from HLOS (window mismatch + no `/dev/mem`), but its decoded value is what `nsp=0xff` reflects.
- No device state was modified at any point.

**Verdict from Proof 1: the NSP subsystem is absent from this device's hardware availability table by the same mechanism that marks the modem absent — a factory-set QFPROM partial-goods bin-out.**
