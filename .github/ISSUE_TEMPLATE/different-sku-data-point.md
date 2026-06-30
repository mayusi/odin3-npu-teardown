---
name: Different-SKU data point
about: Report your Odin 3 / sibling device's NPU/NSP socinfo state (especially if nsp != 0xff)
title: "[DATA] socinfo nsp state on <device / SKU>"
labels: data-point
---

## Device

- Model / SKU (e.g. CQ8725S-3-AA):
- `ro.soc.model` / `chip_id`:
- `soc_id`:
- Firmware build (`image_version`, e.g. eng.Odin3... or a retail build):

## socinfo dump (READ-ONLY — scrub your serial/IMEI first)

Paste the output of:
```sh
adb shell 'for f in nsp npu modem gpu video camera audio spss wlan sku soc_id revision \
  num_subset_parts subset_parts; do echo "$f=$(cat /sys/devices/soc0/$f 2>/dev/null)"; done'
```

```
<paste here — REMOVE serial_number / any IMEI>
```

## CDSP / FastRPC state

```sh
adb shell 'cat /proc/device-tree/soc/remoteproc-cdsp@32300000/status; ls /dev/fastrpc-cdsp'
```

```
<paste here>
```

> **If your `nsp` reads `0x0`** that is a significant finding — it would mean a differently-binned unit exists. Please include as much detail as possible.
