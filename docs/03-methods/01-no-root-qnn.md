# Method 1: No-Root QNN/NPUBench Probing

**Date:** 2026-06-25
**Status:** Confirmed blocked — `NPU_BLOCKED_PLATFORM`
**Evidence:** `evidence/` (see below)

---

## What This Was

The first investigative step was the most accessible: install NPUBench on the Odin 3 without any root preparation, instrument the QNN HTP (Hexagon Tensor Processor) path end-to-end, and observe exactly where the call chain terminates. The goal was to distinguish an app-layer configuration gap from a genuine platform-level block.

Two probe runs were captured:

- `evidence/npubench-no-root-probe/headless-probe.log` — first run, `allowRootPrep=false`, summary classification `FALLBACK_GPU`
- `evidence/npubench-no-root-probe/headless-probe-fixed.log` — second run after delegate config refinement, summary classification `NPU_BLOCKED_PLATFORM`

The second run (fixed) is the definitive verdict.

---

## Device Identity

Both runs confirmed identical device facts:

| Property | Value |
|---|---|
| `manufacturer` | `ayn` |
| `model` | `Odin3` |
| `device` / `board` | `sun` |
| `ro.soc.model` | `CQ8725S` |
| `ro.board.platform` | `sun` |
| Android SDK | 35 (Android 15) |
| ABIs | `arm64-v8a` |

`CQ8725S` is the commercial designation for the Qualcomm SM8750 ("sun") SoC. The `sun` platform tag matches across all firmware artifacts.

---

## The Call Chain

QNN's path to the NPU/HTP goes through FastRPC domain 3, which maps to the CDSP (Compute DSP). The probe log traces the full sequence:

### Step 1 — Library load (succeeds)

```
multimspdsplib_env_init: libcdsprpc.so loaded
dlopen("/vendor/lib64/libcdsprpc.so", RTLD_NOW|RTLD_GLOBAL) SUCCESS
```

`libcdsprpc.so` loads and resolves correctly. This is not a library path or namespace problem.

### Step 2 — FastRPC user-space initialisation (succeeds)

```
fastrpc_apps_user_init done  with default domain:3 and &fastrpc_trace:0x7af04ce640
```

The FastRPC user-space client initialises with domain 3 (CDSP) as its default. The library believes it is ready to connect.

### Step 3 — QNN backend create (succeeds)

```
QnnDsp <I> QnnBackend_create done successfully.  backend = 0xaba9482​8
QnnDsp <I> QnnDevice_create started
```

The QNN DSP backend is created in memory without error. Connection to the "QNN stub" is reported as established.

### Step 4 — Device node open (fails, hard stop)

```
Error 0xe: remote_handle_control_domain failed for request ID 2 on domain 3 (errno No such file or directory)
Error 0x0: open_device_node failed for domain ID 3, sess ID 0 (errno 2, No such file or directory)
(Either the remote processor is down, or the application does not have permission to access the remote processor)
```

The call to `open_device_node` for **domain ID 3** fails with `ENOENT`. The device node `/dev/fastrpc-cdsp` simply does not exist. FastRPC cannot open the CDSP channel.

The probe's own platform evidence block records this explicitly:

```
/dev/fastrpc-cdsp missing
/dev/fastrpc* = fastrpc-adsp-secure
/proc/misc = 107 fastrpc-adsp-secure
cdsprpcd = (empty)
```

Only `fastrpc-adsp-secure` is present. There is no `fastrpc-cdsp` entry in `/proc/misc`. The `vendor.cdsprpcd` service is not running.

### Step 5 — QNN DSP transport failure (cascades from step 4)

```
QnnDsp <E> createUnsignedPD unsigned PD or DSPRPC_GET_DSP_INFO not supported by HTP
QnnDsp <E> DspTransport.createUnsignedPD failed, 0x00000003
QnnDsp <E> Failed to create transport instance: 1002
QnnDsp <E> Transport layer setup failed: 14001
QnnDsp <E> Failed to parse platform config: 14001
QnnDsp <I> QnnDevice_create done.  status 0x36b1
[QNN Delegate] Failed to create device_handle for Backend ID 6, error=14001
```

Error `0x36b1` / `14001` is QNN's platform-config failure code. It propagates up to the TFLite delegate layer, which then restores its original CPU execution plan.

### Step 6 — Final classification

```
NPUBenchProbe: summary classification=NPU_BLOCKED_PLATFORM rows=6
NPUBenchProbe:   route=LITERT_QNN_HTP  class=NPU_BLOCKED_PLATFORM  init=false  npu=false  ran=false  latencyMs=-
NPUBenchProbe:   | platformEvidence: /dev/fastrpc-cdsp missing; /dev/fastrpc* = fastrpc-adsp-secure; ...
```

The `NPU_BLOCKED_PLATFORM` classification is specific: this is not a missing model, wrong permissions, or delegate misconfiguration. The probe distinguishes it from `FALLBACK_GPU` (which the first run returned before the delegate config was refined). On the corrected second run, the full QNN HTP path executes and terminates cleanly at the missing device node.

---

## What Did Run

The non-NPU paths all completed inference successfully:

| Route | Class | Latency |
|---|---|---|
| `LITERT_CPU` | `FALLBACK_CPU` | 672.7 ms |
| `LITERT_GPU` | `FALLBACK_GPU` | 227.3 ms |
| `LITERT_NNAPI` | `UNKNOWN` | 666.3 ms |
| `LITERT_QNN_HTP` | `NPU_BLOCKED_PLATFORM` | — |
| `QNN_DIRECT` | `UNKNOWN` | — (no QNN\_BIN asset) |

The GPU path is fully functional. CPU and NNAPI (CPU fallback) both work. The block is exclusive to the CDSP/HTP path.

---

## What This Rules Out

| Hypothesis | Ruled out by |
|---|---|
| QNN not installed / library missing | `libcdsprpc.so` loads via `dlopen` successfully |
| App namespace or permission issue | `libcdsprpc.so` resolves; FastRPC user init completes |
| QNN version mismatch | Backend create succeeds; transport is the first failure |
| Kernel module missing | `frpc_adsprpc` module is loaded; ADSP domain works |
| SELinux denial | SELinux status was unreadable, but absence of `/dev/fastrpc-cdsp` is kernel-level; SELinux cannot create a device node that the kernel never registered |

---

## The Conclusion

The block is at platform level. The kernel never registers `/dev/fastrpc-cdsp` because the CDSP remoteproc is never brought up. Why the CDSP is never brought up is the subject of the subsequent investigation — see:

- [Method 2: Firmware Inventory](./02-firmware-inventory.md) — firmware exists but is not loaded
- [Method 3: XBL Boot Policy](./03-xbl-boot-policy.md) — the policy decision that suppresses CDSP bringup

> **Note on reproducibility:** The no-root QNN probe is non-destructive. It requires only ADB with USB debugging enabled and NPUBench installed. It does not require root, bootloader unlock, or any firmware modification. The classification is deterministic for this device configuration and will return `NPU_BLOCKED_PLATFORM` on every run until (and unless) the underlying platform block is resolved.
