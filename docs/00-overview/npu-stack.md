# The NPU Stack: How Hexagon, CDSP, FastRPC, and QNN Connect

Understanding why the NPU is inaccessible requires understanding the full stack of layers between the application and the silicon. This page explains that stack — what each layer does, why each layer matters, and what breaks when the layer below it is absent.

---

## The Stack, Bottom to Top

```
┌─────────────────────────────────┐
│  User application / ML runtime  │  ← "I want to run inference"
├─────────────────────────────────┤
│  QNN SDK / HTP backend          │  ← Qualcomm Neural Network SDK
├─────────────────────────────────┤
│  /dev/fastrpc-cdsp              │  ← FastRPC device node
├─────────────────────────────────┤
│  CDSP remoteproc                │  ← Linux kernel subsystem driver
├─────────────────────────────────┤
│  CDSP device-tree node          │  ← DT status="okay" / "no"
├─────────────────────────────────┤
│  QSPA (SoC Platform Arbiter)    │  ← Firmware arbiter
├─────────────────────────────────┤
│  socinfo nsp byte               │  ← 0x0 = present, 0xff = absent
├─────────────────────────────────┤
│  XBL fuse read (QFPROM)         │  ← Bootloader reads OTP at boot
├─────────────────────────────────┤
│  NSP_DISABLE OTP fuse           │  ← ONE-WAY HARDWARE FUSE (silicon)
└─────────────────────────────────┘
```

Every layer collapses if the one below it signals "absent". On the Odin 3, the collapse happens at the very bottom — at the fuse — and propagates upward without stopping.

---

## Layer 1: The OTP Fuse (QFPROM)

**QFPROM** (Qualcomm Fuse Programmable Read-Only Memory) is an array of one-time-programmable fuses on the die. Individual bits can be blown (set to 1) but never unblown. Once set, they survive power cycles, software wipes, and factory resets permanently.

The fuse relevant here is:

```
QFPROM_CORR_FEAT_CONFIG_ROW1_MSB[NSP_DISABLE]
```

This is fuse ID 22 in Qualcomm's internal fuse index, accessed by the FeatureEnabler trustlet via `qsee_is_sw_fuse_blown(fuse_id=22)`. On this device, this fuse reads as **blown**.

The physical QFPROM word that encodes it is at address `0x221c2420`. This address is **outside the HLOS-accessible aperture** — Android's view of QFPROM is restricted to `0x221c8000–0x221c8fff` (from the device-tree `reg` property: `22 1c 80 00 / 00 00 10 00`). The kernel has `CONFIG_DEVMEM` disabled; `/dev/mem` does not exist. Even with root, this word cannot be read directly on-device. The conclusion that it is blown is derived from the behavior of every layer above it — all of which behave as if the fuse is blown.

---

## Layer 2: XBL (eXtensible Bootloader) Fuse Read

XBL is the primary bootloader, signed and sealed with SecureBoot SHA-384. During early boot, XBL reads QFPROM — including the NSP disable fuse word — and publishes the result as a structured socinfo blob that the kernel later maps via `/sys/devices/soc0`.

Specifically: XBL reads QFPROM word `0x221c2420`, masks the low byte (`& 0xff`), and publishes the result as `socinfo.nsp`. On this device that value is `0xff` — the canonical "absent" sentinel used throughout the Qualcomm platform for fused-off features.

Because XBL is SecureBoot-signed, it cannot be replaced or patched on a locked or orange-state device without breaking chain-of-trust. Even with an unlocked bootloader, a replacement XBL would not pass the XBL-level signature check from the SBL (Secondary Bootloader). This path is closed.

---

## Layer 3: Socinfo (`/sys/devices/soc0`)

The kernel's socinfo driver reads the XBL-published blob and exposes per-feature presence bytes as sysfs nodes. On this device:

```
/sys/devices/soc0/nsp  →  ff
```

This is read-only from Android. It cannot be spoofed or patched at runtime without a custom kernel, and even a custom kernel with a patched socinfo driver would only fool software above it — it would not cause the CDSP to actually start.

---

## Layer 4: QSPA (SoC Platform Arbiter) — `qspa.nsp`

QSPA is a Qualcomm firmware component that arbitrates platform-level resource and subsystem policy. It reads the socinfo NSP byte and sets its own internal `qspa.nsp` flag accordingly. When `nsp=0xff`, QSPA marks the NSP subsystem as disabled and will not permit CDSP bring-up.

---

## Layer 5: CDSP Device-Tree Node (`status`)

The Compute DSP (CDSP) is the specific hardware subsystem that hosts the Hexagon processor cores used for NPU/DSP workloads. It is described in the device-tree. When QSPA or the bootloader determines the NSP is absent, the CDSP device-tree node gets its `status` property set to `"no"` (or left absent) — the standard DT mechanism for disabling a peripheral.

On this device, the CDSP node has `status="no"`.

---

## Layer 6: CDSP Remoteproc (Linux Kernel Driver)

The Linux `remoteproc` subsystem manages bring-up of remote processors (modem, ADSP, CDSP, etc.). Each is driven by a kernel driver that loads firmware via the `request_firmware` mechanism, starts the processor, and establishes an IPC channel.

When the CDSP device-tree node has `status="no"`, the remoteproc driver for CDSP is never probed. The CDSP is never started. As a direct consequence, no CDSP firmware image is loaded.

This is confirmed by the debugfs CDSP firmware slots: `cdsp`, `cdsp1`, and `gpdsp` image slots are **empty** — no firmware present, no firmware loaded. Compare this with the ADSP, which is present and shows a populated firmware slot (`CQ8750-ADSP-R01-V1.6`).

---

## Layer 7: `/dev/fastrpc-cdsp`

FastRPC is the IPC mechanism through which Android-side code (including QNN/HTP) invokes code running on the DSP. It exposes a device node per DSP: `/dev/fastrpc-cdsp` for the Compute DSP.

Because the CDSP remoteproc driver was never probed and the CDSP was never started, the `fastrpc-cdsp` device node is never created. It does not exist on this device.

---

## Layer 8: QNN HTP Backend

QNN (Qualcomm Neural Network SDK) uses the HTP (Hexagon Tensor Processor) backend to offload inference to the Hexagon NPU. The HTP backend communicates with the DSP via FastRPC — it opens `/dev/fastrpc-cdsp` and loads a Hexagon DSP library image.

Without `/dev/fastrpc-cdsp`, QNN HTP cannot initialize. The backend fails at open time. This is the user-visible symptom: any attempt to use QNN with the HTP backend on this device fails.

---

## The FeatureEnabler Trustlet: Why Software Cannot Unlock This

A common theory is that Qualcomm's **FeatureEnabler** trustlet — which is known to unlock certain software-licensed features on some devices — could be used to enable the NSP. This theory is wrong for two reasons:

1. **FeatureEnabler does not have an NSP/NPU feature.** Reverse engineering confirms FeatureEnabler only licenses `DisplayCore` features (FPS, resolution, AntiAging, Demura, SPR limiters) and `VideoCore` features (resolution limiter) via RPMB and software fuses. No NSP or NPU feature code exists in the trustlet.

2. **FeatureEnabler itself reads the hardware fuse.** Before licensing any feature, FeatureEnabler calls `qsee_is_sw_fuse_blown(fuse_id=22)` — a TrustZone API that directly checks the `NSP_DISABLE` OTP fuse in QFPROM. If the fuse is blown, FeatureEnabler cannot proceed for NSP regardless of what license key is presented. The hardware check is upstream of the software license.

See [`04-evidence/`](../04-evidence/index.md) for the FeatureEnabler trustlet analysis artifacts.
