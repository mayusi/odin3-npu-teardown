# The HTP Runs Under Linux

**Contributor:** [@WhiteEagle-12](https://github.com/WhiteEagle-12) · **Reported:** 2026-10-05 in [issue #1](https://github.com/mayusi/odin3-npu-teardown/issues/1) · **Impact:** overturns the hardware-fuse verdict

!!! quote "Credit"
    Everything on this page is @WhiteEagle-12's work: the environment, the bring-up, the correctness test, the benchmarks, and the diagnosis of the FastRPC shell trap. It was reproduced here with their numbers and steps so it is preserved alongside the original investigation. Any errors in transcription are mine.

---

## Result

On an Odin 3 with an identical SoC identity — **CQ8725S, `soc_id` 706, revision 2.0, `num_subset_parts` 20**, the same values recorded in this repo — running Linux instead of stock Android, the CDSP boots and the QNN HTP backend executes real graphs correctly at accelerator speed.

**The NSP/HTP silicon is present and functional.** Stock Android never starts the CDSP; a kernel and device tree that bring it up regardless work fine.

## Environment

| Component | Version |
|---|---|
| OS | Holodor (Arch Linux ARM handheld distro) |
| Kernel | `linux-pocknix-sm8750 7.1.3-33`, own device tree; remoteproc loads the CDSP firmware |
| FastRPC | open-source quic/fastrpc v1.0.8, built on-device |
| QNN / QAIRT SDK | 2.42.0.251225 (aarch64 runtime, `hexagon-v79` skels) |
| Unsigned PD shell | `fastrpc_shell_unsigned_3`, copied from the device's own `dsp_a` partition |

## Evidence

### The CDSP boots

```
remoteproc remoteproc1: cdsp is available
remoteproc remoteproc1: Booting fw image qcom/sm8750/ayn/odin3/cdsp.mbn, size 3867048
remoteproc remoteproc1: remote processor cdsp is now up
```

```
/sys/kernel/debug/qcom_socinfo/cdsp/name    = 16:CDSP.HT.3.1-00847.1-PAKALA-1
/sys/kernel/debug/qcom_socinfo/cdsp/variant = pakala.cdsp.prodQ
/sys/class/remoteproc/remoteproc1: cdsp running
/dev/fastrpc-cdsp, /dev/fastrpc-cdsp-secure present
```

Compare with stock Android, where the CDSP socinfo slots are empty and `/dev/fastrpc-cdsp` never appears (see [The Verdict](../01-the-verdict/index.md#the-full-causal-chain)).

### The HTP skel loads

```
Info: Created user PD on domain 3, ... Unsigned:Y, Signed:N ...
remote_handle64_open: Successfully opened handle ... for file:///libQnnHtpV79Skel.so?qnn_skel_handle_invoke&_modver=1.0&_dom=cdsp on domain 3
```

`qnn-platform-validator --backend dsp` reports **Hexagon Architecture V79**.

### Correctness

An ESPCN-style 2× super-resolution network (960×540 → 1920×1080 RGB) was run on the HTP and on the QNN CPU backend with identical input:

```
max abs diff 0.00061, mean abs diff 0.00008, correlation 0.9999997
```

That is FP16-level agreement. The DSP is doing the math, not returning zeros or garbage.

### Performance

`qnn-net-run --profiling_level basic --perf_profile burst`, 20 inferences, average accelerator execute time, 6 HVX threads reported:

| Model / precision | HTP execute time |
|---|---|
| FP32 graph, no backend config | 28 ms |
| FP16 (`fp16_relaxed_precision`) | 5.7 ms |
| INT8 (`qairt-quantizer`, A8W8), 2.7 GMAC | 3.4 ms |
| INT8, 7.8 GMAC (32-channel variant) | 3.5 ms |

The 7.8 GMAC model runs in about the same time as the 2.7 GMAC one: roughly **4.5 TOPS INT8**, and likely bandwidth-bound rather than compute-bound. Whether HMX or only HVX was used is not yet confirmed.

## Why it looked disabled

1. **Stock Android never starts the CDSP.** The gating lives in the stock boot/vendor configuration, not in the silicon.
2. **A FastRPC trap that makes a Linux attempt look disabled too.** quic/fastrpc only looks for `fastrpc_shell_unsigned_3` in the per-machine YAML paths (`<config_base_dir>/conf.d/*.yaml`), then falls back to `/usr/lib/rfsa/adsp`, `/usr/lib/dsp` and `/vendor/dsp`. It ignores `ADSP_LIBRARY_PATH` for the shell. Without a matching YAML entry, every HTP open fails with `0x80000580` (`remote_init failed`) — easy to misread as the DSP being unavailable.

## What this changes in the original analysis

| Original proof | Status after this finding |
|---|---|
| `soc0/nsp = 0xff` from the `subset_parts` table | The value is real, but it does not mean absent silicon on this part. XBL fills this table before any OS runs, so it most likely reads `0xff` under Linux too — it is a label the stock Android stack obeys, not a kill switch. |
| FeatureEnabler `qsee_is_sw_fuse_blown(22)` | Needs re-review. The trustlet never touches the QFPROM aperture (see [Proof 2](../02-how-we-proved-it/proof-2-featenabler.md)); mapping fuse 22 to a QFPROM OTP bit was an inference, and QSEE "sw fuses" may be TZ-maintained flags. |
| Patent US 12,061,855 | Shows the harvesting mechanism exists. It does not show it was applied to this part. |
| Fuse aperture not mapped, `/dev/mem` off | Still true, and still why Android-side probing could not see past the policy. |

The Android-side findings remain accurate as documentation of **how** the stock firmware keeps the CDSP down. The conclusion that the disable is a permanent hardware fuse does not hold.

## Reproduce (Linux on the Odin 3, as root)

1. Build quic/fastrpc with `--with-config-base-dir=<dir>` and add `<dir>/conf.d/odin3.yaml`:
   ```yaml
   machines:
     AYN Odin 3:
       DSP_LIBRARY_PATH: dsp/
       CDSP_ARCH: v79
   ```
2. Put `fastrpc_shell_unsigned_3` from the `dsp_a` partition in `<dir>/dsp/cdsp/`.
3. Set `LD_LIBRARY_PATH` to the QNN aarch64 libs and the FastRPC build, and set `ADSP_LIBRARY_PATH` to the `hexagon-v79` skel directory.
4. Run:
   ```
   qnn-net-run --backend libQnnHtp.so --model libQnnModelDlc.so --dlc_path model.dlc --input_list list.txt
   ```

## Open questions

- **The `nsp` byte under Linux.** Mainline socinfo doesn't expose per-part names, so it hasn't been read on this kernel. Dumping the raw socinfo SMEM would settle what `0xff` means here.
- **Android.** Whether a modified Android boot chain can do the same is untested. Given the [dm-verity brick](../03-methods/08-brick-and-recovery.md) documented in this repo, don't try it casually.
- **HMX vs HVX.** Not yet confirmed which units the graphs ran on.
