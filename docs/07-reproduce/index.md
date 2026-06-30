# Reproducing the Key Findings

These are read-only verification steps that any Odin 3 owner with `adb` can run to confirm the investigation's primary conclusions. **No root is needed for the socinfo checks.** Root is needed only for the FastRPC and device-tree checks noted below.

---

> **Warning: Do NOT flash anything.**
>
> This page describes **read-only** commands only. There are no write operations here, no partition modifications, no kernel module loads.
>
> During this investigation, a modified `vendor_dlkm` image was flashed and the device bricked: the signed `vbmeta_a` partition's verity digest did not match the modified partition, causing `veritymode=eio` on every boot. Full EDL recovery was required.
>
> If you are following this guide to reproduce findings, you should never need to write to the device. If any guide, forum post, or script tells you to flash anything in pursuit of NPU access, stop — the analysis on this page proves there is nothing to unlock.
>
> See [`evidence/avb-verity/`](https://github.com/mayusi/odin3-npu-teardown/tree/main/evidence/avb-verity) and [`evidence/vendor-dlkm-postmortem/`](https://github.com/mayusi/odin3-npu-teardown/tree/main/evidence/vendor-dlkm-postmortem) for the brick postmortem.

---

## Prerequisites

- AYN Odin 3 with USB debugging enabled.
- `adb` installed on your host machine.
- For steps marked **[root required]**: a rooted Odin 3 (bootloader unlocked, root method active). Root is NOT needed for the socinfo reads.

Connect your device and verify ADB access before starting:

```sh
adb devices
# Expected: your device listed as "device" (not "unauthorized")
```

---

## Check 1 — Read the NSP / NPU / Modem Binning Flags (No Root Required)

The `soc0` sysfs interface is world-readable. This is the most important single check.

```sh
adb shell cat /sys/devices/soc0/nsp
adb shell cat /sys/devices/soc0/npu
adb shell cat /sys/devices/soc0/modem
```

**Expected output on an Odin 3 (CQ8725S):**

```
nsp:   255     ← 0xff in decimal: binned-out sentinel
npu:   0
modem: 255     ← 0xff: modem is also absent on the CQ8725S IoT SKU
```

**What these values mean:**

- `nsp = 255` (`0xff`): XBL read the QFPROM `NSP_DISABLE` fuse at boot and published `0xff` to the `subset_parts` (formerly `defective_parts`) table, signalling the NSP subsystem is absent for this chip.
- `npu = 0`: Most likely indicates the NPU/AI-accelerator block is not flagged as binned out (the exact per-SoC field meaning is inferred — SM8750 `subset_parts` mapping is proprietary CLO, not in upstream `socinfo.c`). In any case, it is not an independently addressable subsystem; the NSP bring-up is what makes it usable, and NSP is disabled.
- `modem = 255` (`0xff`): The modem subsystem is also binned out on CQ8725S. This validates the sentinel encoding: on a phone-class SM8750 device (`modem = 0`), you would see a different value.

**Comparison on a phone-class SM8750 (e.g., OnePlus 13):**

```
nsp:   0       ← NSP present and enabled
npu:   0       ← same field
modem: 0       ← modem present
```

The `0xff` sentinel on the Odin 3 is device-specific and driven by the QFPROM fuse content that XBL read.

---

## Check 2 — Confirm /dev/fastrpc-cdsp Is Absent (No Root Required)

CDSP (Compute DSP) is the Hexagon DSP core that the NPU runs through. If it were active, a FastRPC device node would exist.

```sh
adb shell ls /dev/fastrpc*
```

**Expected output on an Odin 3:**

```
/dev/fastrpc-adsp
/dev/fastrpc-mdsp  (may or may not be present)
# /dev/fastrpc-cdsp is NOT listed
```

**What this means:** FastRPC domain 3 (CDSP) is not registered. The remoteproc subsystem never brought CDSP up because XBL set the CDSP device-tree node to `status = "disabled"` based on the fuse read.

**Cross-check:** On a phone-class SM8750 with NPU enabled, `/dev/fastrpc-cdsp` would exist.

---

## Check 3 — Confirm the QFPROM HLOS Window Does Not Cover the Fuse Address (No Root Required)

The HLOS-mapped QFPROM nvmem window is visible in `/proc/iomem`. You can confirm that address `0x221c2420` — the fuse register XBL reads to set the NSP flag — is outside this window.

```sh
adb shell grep -i qfprom /proc/iomem
```

**Expected output (representative):**

```
221c8000-221c8fff : /soc/qfprom@221c8000
```

**What to look for:** The HLOS window starts at `0x221c8000`. The fuse address XBL reads is `0x221c2420` — which is `0x5BE0` bytes *below* the HLOS window start. It is not reachable from Android, even with root.

This explains why a direct fuse read is not possible from userspace: the QFPROM aperture that HLOS can access is intentionally windowed to exclude the raw corrected fuse bank.

---

## Check 4 — Confirm CDSP Device-Tree Status [Root Required]

The device-tree blob that XBL prepared can be read at runtime. This confirms XBL explicitly disabled the CDSP node before handing control to the kernel.

```sh
# With root:
adb shell su -c "cat /sys/firmware/devicetree/base/soc/remoteproc@2c00000/compatible"
adb shell su -c "cat /sys/firmware/devicetree/base/soc/remoteproc@2c00000/status"
```

**Expected output:**

```
qcom,sm8750-cdsp-pas
disabled
```

**What this means:** The CDSP remoteproc node is present in the device tree (the hardware is physically there) but `status = "disabled"`, which causes the Linux `qcom_q6v5_pas` driver to skip the entire bring-up sequence. No firmware is loaded, no power domain is enabled, no PAS registration occurs.

> **Note on path:** The exact DT node address (`2c00000`) is correct for SM8750. If your device tree layout differs, search for `cdsp` nodes:
> ```sh
> adb shell su -c "find /sys/firmware/devicetree/base -name 'compatible' | xargs grep -l cdsp 2>/dev/null"
> ```

---

## Check 5 — Confirm /dev/mem Is Unavailable (No Root Required)

Some approaches to reading arbitrary physical memory rely on `/dev/mem`. Confirm it is compiled out:

```sh
adb shell ls -la /dev/mem 2>&1
```

**Expected output:**

```
ls: /dev/mem: No such file or directory
```

**What this means:** `CONFIG_DEVMEM` is not set in the Odin 3 kernel. There is no path to map physical addresses (including QFPROM) from userspace via `/dev/mem`.

---

## Summary of Expected Results

| Check | Command (abbreviated) | Expected Result | Root? |
|-------|----------------------|----------------|-------|
| 1a | `cat /sys/devices/soc0/nsp` | `255` (`0xff`) | No |
| 1b | `cat /sys/devices/soc0/npu` | `0` | No |
| 1c | `cat /sys/devices/soc0/modem` | `255` (`0xff`) | No |
| 2 | `ls /dev/fastrpc*` | `/dev/fastrpc-cdsp` absent | No |
| 3 | `grep -i qfprom /proc/iomem` | Window at `0x221c8000` only | No |
| 4 | CDSP DT node status | `disabled` | Yes |
| 5 | `ls /dev/mem` | No such file | No |

All five checks are read-only. None of them require writing to the device, modifying any partition, or loading any kernel module. The results are stable across reboots and firmware versions, because they are set by XBL from hardware state before Android loads.

---

## If Your Results Differ

If you see `nsp = 0` (instead of `255`) on your Odin 3, that would be significant — please open an issue in this repository with:

- Full `adb shell cat /sys/devices/soc0/*` output.
- Output of `adb shell getprop ro.product.model` and `ro.product.board`.
- Output of `adb shell cat /proc/version`.

Do **not** attempt to flash anything based on speculation about what your results might mean. Gather the read-only evidence first.
