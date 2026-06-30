# Recovery After Vendor DLKM Failure Verdict

Label: RECOVERED_AFTER_FULL_STOCK_SUPER_RESTORE_AND_MANUAL_UNLOCK_WARNING_CONTINUE

## Confirmed Live State
- Device: <SERIAL>
- Android: boot_completed=1
- Slot: _a
- Verified boot state: orange (expected unlocked warning)
- Verity mode: eio
- NSP: 0xff
- QSPA NSP: disabled
- CDSP FastRPC: NO_FASTRPC_CDSP
- /vendor_dlkm: mounted read-only from dm-18
- Live qcom_q6v5_pas.ko SHA-256: d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9
- Stock qcom_q6v5_pas.ko SHA-256: d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9

## Interpretation
The phone is back in Android. The confusing screen was the normal unlocked/orange warning path; pressing Power continued boot. The recovery action that mattered was the completed fastbootd full stock super restore. The boot-critical restore attempt should not be treated as completed because fastboot later reported Write to device failed (no link).

## Decision
The candidate vendor_dlkm path is retired for now as boot-breaking or verity/metadata incompatible. No more live vendor_dlkm candidate flashes until offline root cause is proven and a new destructive checkpoint is approved.
