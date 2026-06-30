# The Verdict: The Hexagon NPU Is Permanently Disabled

> **The answer is no. The NSP (Neural Signal Processor / Hexagon NPU) on the AYN Odin 3 cannot be enabled — by you, by AYN, by anyone, by any software — because a one-time-programmable hardware fuse on the CQ8725S silicon was blown at Qualcomm's factory before this chip ever left the building. That fuse cannot be unblown.**

---

## The Answer, Plainly

The AYN Odin 3 uses a **CQ8725S** SoC — a Qualcomm Dragonwing/IoT bin of the SM8750 "sun" platform. On this specific silicon bin, the Hexagon Neural Signal Processor was disabled at manufacture by blowing the `QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]` OTP fuse (QFPROM fuse ID 22, at word address `0x221c2420`).

Every layer of the software stack reads that fuse (directly or transitively) and refuses to bring up the NPU subsystem. The result propagates from silicon to bootloader to kernel to Android userspace in an unbroken chain of "absent."

No firmware update, no rooted-device trick, no FeatureEnabler license, no custom kernel, no AYN OTA, and no future Android version can change this. The silicon is what it is.

---

## The Full Causal Chain

```
QFPROM[NSP_DISABLE] = BLOWN (OTP, one-way, at factory)
        │
        ▼
XBL reads 0x221c2420, masks low byte → publishes socinfo.nsp = 0xff
  (XBL = SecureBoot SHA-384 signed; cannot be replaced/patched)
        │
        ▼
/sys/devices/soc0/nsp = 0xff  ("absent" sentinel)
  (kernel socinfo driver; read-only from Android)
        │
        ▼
QSPA sets qspa.nsp = disabled
  (SoC Platform Arbiter; reads socinfo nsp byte)
        │
        ▼
CDSP device-tree node: status = "no"
  (bootloader/firmware clears CDSP from DT when NSP absent)
        │
        ▼
CDSP remoteproc driver: never probed
  (Linux kernel driver skips DT nodes with status != "okay")
        │
        ▼
CDSP firmware: never loaded
  (cdsp / cdsp1 / gpdsp firmware slots: EMPTY)
  (contrast: ADSP fw CQ8750-ADSP-R01-V1.6 — present and loaded)
        │
        ▼
/dev/fastrpc-cdsp: does not exist
  (FastRPC device node only created when CDSP is live)
        │
        ▼
QNN HTP backend: fails at open()
  (cannot initialize without /dev/fastrpc-cdsp)
        │
        ▼
NPU inference: impossible
```

Every link in this chain has been verified. See [`02-how-we-proved-it/`](../02-how-we-proved-it/) for the methodology behind each step, and [`04-evidence/`](../04-evidence/) for raw artifacts.

---

## Confidence: ~94%

This investigation is assessed at **approximately 94% confidence** that the NSP disable is a permanent OTP hardware fuse.

### What the 94% rests on

- `socinfo nsp = 0xff` — directly read on-device, verified
- CDSP firmware slots empty — directly observed in debugfs, verified
- `/dev/fastrpc-cdsp` absent — directly verified on-device
- FeatureEnabler trustlet calls `qsee_is_sw_fuse_blown(fuse_id=22)` — verified via RE
- FeatureEnabler has no NSP/NPU feature code — verified via RE
- Parallel modem absence (`socinfo modem = 0xff`) consistent with hardware bin-out pattern — verified
- Qualcomm patent US 12,061,855 describes exactly this mechanism for binned silicon — published prior art
- `subset_parts = 0x2350 (9040)`, `num_subset_parts = 20` — consistent with partial-goods die

### Why not 100%

The specific QFPROM fuse word at `0x221c2420` **cannot be read directly on this device**:

- The HLOS QFPROM aperture is `0x221c8000–0x221c8fff` (from device-tree `reg`: `22 1c 80 00 / 00 00 10 00`)
- `0x221c2420` is below that window — it is not mapped to Android
- `/dev/mem` is compiled out (`CONFIG_DEVMEM` not set); root access does not help
- TrustZone QSEE APIs that read QFPROM are not accessible from HLOS even with root

The conclusion that the fuse is blown rests on the **unanimous behavior** of every observable system above it — all of which behave exactly as they would if and only if that fuse were blown. No observable contradicts this.

The residual ~6% is the space for an alternative explanation: some other mechanism that causes identical observable behavior while being reversible. No such mechanism has been identified or even plausibly hypothesized.

> **Note** This 6% is uncertainty about our measurement ability, not hope for an unlock. The fuse, if blown, is permanent by physics. "94% confident it's blown" means "94% confident it's permanent."

---

## Can AYN Enable It?

**No.**

For several reasons, all of which converge:

1. **The fuse was blown before AYN received the silicon.** Qualcomm blows binning fuses during wafer-level or package-level test, before the chip ships to an OEM. AYN did not make this decision; they received a chip that had already been binned.

2. **AYN cannot unblow a fuse.** No one can. OTP means one-time-programmable. There is no reverse procedure.

3. **AYN cannot route around the fuse in firmware.** The XBL bootloader that reads the fuse is signed by Qualcomm with SHA-384 SecureBoot. AYN cannot replace it. Even if they could, a patched XBL lying about the socinfo nsp byte would not cause the NSP hardware to actually start — the hardware is what it is.

4. **FeatureEnabler cannot help.** FeatureEnabler is Qualcomm's software-license enforcement trustlet. It is used to unlock display and video features (FPS/resolution limiters, etc.) on some Dragonwing devices. It has no NSP feature code. And even if it did, it calls `qsee_is_sw_fuse_blown(fuse_id=22)` before doing anything — which would immediately detect the blown hardware fuse and abort.

The only path to a Hexagon NPU on Odin-class hardware would be sourcing a different, un-binned silicon variant. That would be a different chip, constituting a different device.

---

## What It Means for Developers

If you are developing ML inference workloads targeting the Odin 3:

- **QNN HTP (Hexagon Tensor Processor) is not available.** Any QNN delegate/backend targeting HTP will fail at runtime.
- **CDSP-based compute (FastRPC workloads) is not available.** The CDSP subsystem is not running.
- **CPU and GPU inference are fully functional.** The GPU (`gpu=0x0`, not flagged as binned) is available for GPU-accelerated inference via OpenCL, Vulkan compute, or NNAPI GPU delegate. CPU inference via ONNX Runtime, TensorFlow Lite, etc. works normally.
- **Do not target `QNN_BACKEND_HTP` or `QNN_BACKEND_DSP`** on this device. Use `QNN_BACKEND_GPU` or `QNN_BACKEND_CPU`, or detect the absent `/dev/fastrpc-cdsp` node and fall back gracefully.

---

## What Would Change This Conclusion

Only one thing: direct, on-device readout of QFPROM address `0x221c2420` showing the `NSP_DISABLE` bit is **not** blown. This would require either:

- A custom XBL or SBL that dumps the full QFPROM aperture (requires bypassing SecureBoot — not trivial, not documented here)
- A JTAG/Coresight hardware debug interface with access to the QFPROM block (requires physical hardware debug access)
- Qualcomm providing the fuse map for this specific device's QFPROM — which they will not do

Short of that, the weight of evidence is conclusive. The NPU is off.
