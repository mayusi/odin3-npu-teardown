# Method 2: Firmware Surface Inventory and CDSP Probe

**Date:** 2026-06-28
**Status:** Firmware present — CDSP never brought up, DT status `no`, `nsp=0xff`
**Evidence dirs:**
- `evidence/cdsp-firmware-crosscheck-20260628/`
- `evidence/trustzone-cdsp-probe-20260628/`

---

## Purpose

After the no-root QNN probe confirmed `/dev/fastrpc-cdsp` is absent, the next question was: **is the CDSP firmware even there?** If the image files were missing, the block would be a simple packaging omission. If they are present but never loaded, the block is a policy decision.

This was answered by two parallel probes executed over ADB:

1. A **CDSP firmware crosscheck** — enumerate the actual image files on the device, confirm their presence and sizes.
2. A **TrustZone CDSP probe** — read sysfs, remoteproc state, DT nodes, and QSPA properties to understand what the kernel and firmware policy know about CDSP.

No root was available; the firmware crosscheck used a privileged shell path (`pserver`) where available, and sysfs reads were non-destructive.

---

## QSPA Property State

The QSPA (Qualcomm SoC Platform Attribute) service publishes per-feature enablement decisions at boot. This is the authoritative runtime view:

```
ro.boot.vendor.qspa.nsp     = disabled
ro.boot.vendor.qspa.npu     = enabled
ro.boot.vendor.qspa.comp    = enabled
ro.boot.vendor.qspa.eva     = enabled
ro.boot.vendor.qspa.audio   = enabled
ro.boot.vendor.qspa.gpu     = enabled
ro.boot.vendor.qspa.display = enabled
init.svc.vendor.cdsprpcd    = (empty)
init.svc.vendor.dspservice  = running
init.svc.vendor.qspa-service = running
```

`nsp` is **disabled**. Every other major compute block is enabled. `cdsprpcd` — the daemon that manages the CDSP FastRPC channel and is a prerequisite for `/dev/fastrpc-cdsp` to appear — has an empty service state, meaning it was never started.

The `npu` property being `enabled` while `nsp` is `disabled` is significant: on SM8750, the NPU runs *on* the CDSP/NSP (Neural Signal Processor) fabric. Enabling "npu" without "nsp" is a contradictory policy state that produces exactly the outcome observed — the QNN HTP backend thinks NPU capability is advertised but the CDSP transport is absent.

---

## Device Tree: CDSP Node

The CDSP remoteproc is present in the device tree and is fully described:

```
node: /sys/firmware/devicetree/base/soc/remoteproc-cdsp@32300000
status     = no
firmware   = cdsp.mdt  cdsp_dtb.mdt
compatible = qcom,sun-cdsp-pas
reg        = 00003032 00000100
```

`status = no` is the critical field. In Qualcomm DT convention, a node with `status = no` is explicitly disabled — the kernel will not bind a driver to it, meaning the CDSP PAS (Peripheral Authentication Service) driver never loads, the remoteproc is never registered, and the firmware image is never loaded from disk. The node exists; it is just told not to start.

The `remoteproc` sysfs confirms this: `/sys/class/remoteproc/` lists `remoteproc0` (ADSP), `remoteproc1` (SOCCP), and `remoteproc2` (SPSS). No CDSP remoteproc entry appears.

---

## Firmware Files Present

Despite the disabled DT node, the CDSP firmware image set is fully present on the device under `/vendor/firmware_mnt/image/`:

| File | Size (bytes) | Timestamp |
|---|---:|---|
| `cdsp.mdt` | 5,084 | 2025-12-06 18:45 |
| `cdsp.b00` | 564 | 2025-12-06 18:45 |
| `cdsp.b01` | 27,644 | 2025-12-06 18:45 |
| `cdsp.b02` | 79,076 | 2025-12-06 18:45 |
| `cdsp.b03` | 140,303 | 2025-12-06 18:45 |
| `cdsp.b04` | 1,376 | 2025-12-06 18:45 |
| `cdsp.b05` | 2,411,972 | 2025-12-06 18:45 |
| `cdsp.b06` | 114,696 | 2025-12-06 18:45 |
| `cdsp.b07` | 2,840 | 2025-12-06 18:45 |
| `cdsp.b08` | 124,406 | 2025-12-06 18:45 |
| `cdsp.b09` | 786,556 | 2025-12-06 18:45 |
| `cdsp.b10` | 14,016 | 2025-12-06 18:45 |
| `cdsp.b11` | 3,184 | 2025-12-06 18:45 |
| `cdsp.b12` | 114,096 | 2025-12-06 18:45 |
| `cdsp.b13` | 5,600 | 2025-12-06 18:45 |
| `cdsp.b15` | 4,520 | 2025-12-06 18:45 |
| `cdsp_dtb.mdt` | 4,044 | 2025-12-06 18:45 |
| `cdsp_dtb.b00` | 148 | 2025-12-06 18:45 |
| `cdsp_dtb.b01` | 29,916 | 2025-12-06 18:45 |
| `cdsp_dtb.b02` | 3,896 | 2025-12-06 18:45 |
| `cdspr.jsn` | 513 | 2025-10-22 16:41 |

The `.mdt` manifest, all loadable segments (`.b00`–`.b15`), the device-tree blob set (`cdsp_dtb.*`), and the resource JSON (`cdspr.jsn`) are present. This is a complete firmware package. The 2025-12-06 timestamps are consistent with the shipping AYN firmware build.

---

## `xbl_config` RetailImages: The Smoking Gun

The CDSP firmware files exist, but the boot loader's `RetailImages` list — which tells XBL which images to verify and load as part of the secure boot sequence — does not include CDSP:

**Stock `RetailImages`:**
```
ABL, SPSS, ImageFv, FULL_ADSP, FULL_ADSP_DTB
```

**Missing:**
```
FULL_CDSP, FULL_CDSP_DTB
```

Meanwhile, the CDSP *config nodes* (`FULL_CDSP_CFG`, `FULL_CDSP_DTB_CFG`, `CORE_CDSP_CFG`, `CORE_CDSP_DTB_CFG`) are all present inside `xbl_config`. The config describes the CDSP; the boot list does not include it. This inconsistency is intentional — it is what "disabled by policy" looks like in this firmware architecture.

---

## sysfs Evidence: NSP Value

```
/sys/devices/soc0/nsp = 0xff
```

The `nsp` socinfo field is `0xff`. This value is read by XBL from QFPROM address `0x221c2420` and published as socinfo part 15. `0xff` means "all NSP cores disabled." The QSPA service reads this and sets `ro.boot.vendor.qspa.nsp = disabled`.

The full causal chain:

```
QFPROM[0x221c2420] → XBL reads → nsp = 0xff
  → socinfo part 15 published
  → QSPA reads socinfo
  → ro.boot.vendor.qspa.nsp = disabled
  → cdsprpcd not started
  → /dev/fastrpc-cdsp never created
  → QNN HTP open fails (ENOENT, domain 3)
  → NPU_BLOCKED_PLATFORM
```

---

## TrustZone / Kernel Infrastructure

The kernel-level CDSP infrastructure is present but idle:

- `qcom_q6v5_pas` and `qcom_q6v5` modules are loaded (the Hexagon remoteproc driver)
- `cdsprm` module is loaded (`msm_cdsp_rm` platform driver bound)
- SMMU context banks for CDSP are registered under the ADSP path (IDs 1–6) — these service the ADSP, not an active CDSP
- `/sys/kernel/debug/qcom_socinfo/cdsp` and `/cdsp1` entries exist but represent disabled units
- `glink_pkt_data_cdsp` and `glink_pkt_ctrl_cdsp` glink packet devices are registered, confirming the inter-processor communication bus topology is defined — but no CDSP-side peer will be present since the processor never starts

The kernel is fully prepared to run the CDSP. The DT `status = no` and the missing `RetailImages` entry are the only things in the way — and both trace back to the boot-time policy imposed by XBL reading `0x221c2420`.

---

## Summary

| Component | State |
|---|---|
| CDSP firmware files on device | Present and complete |
| CDSP DT node | Present, `status = no` |
| CDSP remoteproc registered | No |
| `cdsprpcd` service | Never started |
| `/dev/fastrpc-cdsp` | Absent |
| `ro.boot.vendor.qspa.nsp` | `disabled` |
| `/sys/devices/soc0/nsp` | `0xff` |
| `RetailImages` includes CDSP | No |

The firmware is present. The hardware is capable. The block is a policy decision enforced at boot by XBL and published through QSPA. The next step was to characterise that policy and understand whether it is derived from a signed configuration blob or from a hardware fuse. That analysis is in [Method 3: XBL Boot Policy](./03-xbl-boot-policy.md).
