# 06 · Kernel Build & CDSP Probe

> **Status:** `BUILD_READY_NO_FLASH`
> The OnePlus SM8750/sun kernel compiles cleanly with log-only CDSP probe patches.
> The kernel cannot override the boot policy that disables CDSP before Linux binds.
> Flashing a patched `vendor_dlkm` to exercise these modules led to the brick
> described in [Method 08](08-brick-and-recovery.md).

---

## The Plan

If XBL gates NSP via a QFPROM fuse value read before Linux starts, and that gate
cannot be bypassed in firmware, perhaps the Linux kernel itself could probe the CDSP
remoteproc from userspace — collecting evidence about whether CDSP would start if the
DT status node were overridden. A log-only patch would add diagnostic output without
attempting to force hardware that might not be powered.

The goal was not to force the NPU alive. It was to capture what the kernel *would* say
if asked, then compare those logs against donors running the same driver stack.

---

## Source Selection

OnePlus publishes the full kernel source for their SM8750 devices (OnePlus 13,
platform `sun`) under the standard Android kernel open-source program. The same
BSP is used across OEMs; the Odin 3's kernel is a vendor-modified derivative of
this tree.

```
repo init -u https://github.com/OnePlusOSS/kernel_manifest.git \
  -b oneplus/sm8750 -m oneplus_13_b.xml --depth=1
repo sync -c --no-tags --optimized-fetch --prune -j4
./kernel_platform/oplus/build/oplus_build_kernel.sh sun perf
```

Build workspace: WSL2 ext4 volume (not `/mnt/c` — build is too large for the Windows
FS layer). Guide: `evidence/aime-npu-unlocker-finish-20260629/BUILD_AND_PROBE_GUIDE.md`.

---

## Active DT State: The First Blocker

Before any patches, the live device tree was read from the running Odin 3:

```
/sys/firmware/devicetree/base/soc/remoteproc-cdsp@32300000
  compatible = "qcom,sun-cdsp-pas"
  status     = "no"
  firmware-name = "cdsp.mdt, cdsp_dtb.mdt"
```

The node exists and is **complete** — it has all the expected properties:
`clock-names`, `clocks`, `cx-supply`, `mx-supply`, `nsp-supply`, `memory-region`,
`glink-edge`, `interconnects`, `interrupts-extended`, `smem-states`, `qcom,qmp`.
A fully-wired CDSP node with every dependency present. The only thing wrong is
`status = "no"`.

That single string stops the `qcom_q6v5_pas` driver from binding at probe time.
No binding → no CDSP boot → no FastRPC → no `/dev/fastrpc-cdsp` → no NPU path.

**First blocker:** `active DT status=no on qcom,sun-cdsp-pas`

---

## Live Read-Only Device State

Taken from the Odin 3 before any build or flash attempt:

| Property | Value |
|----------|-------|
| `adb_state` | `device` |
| `/sys/devices/soc0/nsp` | `0xff` |
| `qspa.nsp` | `disabled` |
| `qspa.npu` | `enabled` |
| `/dev/fastrpc-cdsp` | `ABSENT` |
| `cdsp_status` | `no` |

`qspa.npu=enabled` is a red herring — it refers to the NPU *property* being
acknowledged, not the subsystem being active. The decisive values are `nsp=0xff`
(fuse reads disabled), `qspa.nsp=disabled`, and `fastrpc_cdsp` absent.

---

## Modules Targeted by the Patch

The probe patch set produced log-only modifications to four kernel modules:

| Module | Role | Source file |
|--------|------|-------------|
| `qcom_q6v5_pas.ko` | CDSP remoteproc / PIL loader | `drivers/remoteproc/qcom_q6v5_pas.c` |
| `socinfo.ko` | Exposes `/sys/devices/soc0/nsp`; reads SOCINFO_PART_NSP | `drivers/soc/qcom/socinfo.c` |
| `frpc-adsprpc.ko` | FastRPC — the userspace DSP RPC mechanism | `vendor/qcom/opensource/dsp-kernel/dsp/fastrpc.c` |
| `cdsp-loader.ko` | High-level CDSP boot sequencer (calls `rproc_boot`) | `vendor/qcom/opensource/dsp-kernel/dsp/cdsp-loader.c` |

The patch added diagnostic `pr_info` / `dev_dbg` calls at probe entry, status-node
evaluation, and failure paths. **No force-enable logic was added.** The `aime.cdsp_force=1`
boot parameter was explicitly excluded at this stage.

The build completed with `BUILD_SUCCESSFUL` and all four target `.ko` files were
confirmed built with:

- **vermagic:** `6.6.87-android15-8-maybe-dirty-4k SMP preempt mod_unload modversions aarch64`
- **modversions CRCs:** all shared `__versions` entries matching the stock kernel
- **No module signature enforcement** (`module.sig_enforce` absent on stock vendor modules)

---

## Why the Kernel Cannot Override Boot Policy

The boot policy trace established a multi-layer disable chain that the kernel
cannot break from Linux-land:

```
XBL reads QFPROM 0x221c2420 → value = 0xff
  → feature_word[15] set in socinfo smem
    → qspa reads feature_word[15]
      → qspa.nsp = disabled
        → cdsprpcd (userspace daemon) stops before CDSP is bootable
          → DT node status=no (set by ABL/bootloader before kernel)
            → qcom_q6v5_pas driver never probes
```

Every step upstream of `status=no` happens before the Linux kernel is handed
control. The QFPROM read fires in XBL (the secondary bootloader). The SMEM
feature word is written before ABL. QSPA reads it from SMEM during early init.
ABL writes `status = "no"` into the flattened device tree passed to the kernel.

By the time `qcom_q6v5_pas.c` tries to probe `qcom,sun-cdsp-pas`, the decision
is already made and baked into the DT the kernel received. Overriding it from
a kernel module would require:

1. Modifying the compiled DTB on the device (requires signed `dtbo` or `vendor_boot`), or
2. Patching ABL (signed, hash-covered), or
3. Getting a different QFPROM value to the XBL rule — which is the root problem.

**Classification:** `BOOTLOADER_OR_BOOT_POLICY_PATCHES_DT_STATUS_AND_QSPA_BOOTCONFIG`

The kernel build is a dead end *for enabling* CDSP. What it *could* still provide,
if a patched `vendor_dlkm` were safely loaded, is the first raw log trace of the
driver stack's response to that DT state. That hypothesis is what the vendor_dlkm
probe in [Method 07](07-vendor-dlkm-probes.md) attempted to test.

---

## What the Source Confirmed About NSP Consumers

The socinfo source search found `SOCINFO_PART_NSP` referenced at 12 call sites
across `include/soc/qcom/socinfo.h` and `drivers/soc/qcom/socinfo.c`:

```c
// include/soc/qcom/socinfo.h:93
SOCINFO_PART_NSP,

// drivers/soc/qcom/socinfo.c
u32 num_subset_parts;
u32 nsubset_parts_array_offset;
socinfo_get_subpart_info(part_enum, part_info, num_parts);
```

The socinfo driver reads the NSP subset-parts array from SMEM and exposes it
at `/sys/devices/soc0/nsp`. This confirms the `0xff` reading on the Odin is not
a kernel bug — the kernel is faithfully reporting the value XBL deposited in SMEM.
Zero NSP consumers (QNN, SNPE, HTP) were found gating on a kernel-level override.
The block is upstream.

---

## Summary

| Label | Meaning |
|-------|---------|
| `BOOT_POLICY_TRACE_BUILT` | Full disable chain traced from XBL through QSPA to DT |
| `ACTIVE_DT_CDSP_NODE_MAPPED` | CDSP DT node confirmed complete but `status=no` |
| `CDSP_DEPENDENCY_CHAIN_MAPPED` | All driver dependencies mapped in source |
| `LOG_ONLY_PATCH_DRAFT_READY` | Patch compiles; no force-enable behavior added |
| `BUILD_READY_NO_FLASH` | Modules built; cannot override pre-Linux boot policy |
| `NSP_CONSUMER_NOT_FOUND` | No kernel-level NSP gate that a module override could flip |

> **Evidence files:**
> `evidence/aime-kernel-last-mile-20260629/LAST_MILE_SUMMARY.json`
> `evidence/aime-kernel-last-mile-20260629/BOOT_POLICY_TRACE.md`
> `evidence/aime-kernel-last-mile-20260629/ACTIVE_DT_CDSP_MAP.md`
> `evidence/aime-npu-unlocker-finish-20260629/BUILD_AND_PROBE_GUIDE.md`
