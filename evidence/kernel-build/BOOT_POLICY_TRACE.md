# Aime Last-Mile Boot Policy Trace

Generated: 2026-06-29T14:32:01

Acceptance: `BOOT_POLICY_TRACE_BUILT`

## Current Read-Only State

| field | value |
| --- | --- |
| adb_state | device |
| nsp | 0xff |
| qspa_nsp | disabled |
| qspa_npu | enabled |
| fastrpc_cdsp | ABSENT |
| cdsp_status | no |


## Active CDSP DT Signal

- Node: `/sys/firmware/devicetree/base/soc/remoteproc-cdsp@32300000`
- Status: `no`
- Compatible: `qcom,sun-cdsp-pas`
- Firmware names: `cdsp.mdt, cdsp_dtb.mdt`
- Present dependencies: `clock_names, clocks, cx_supply, mx_supply, nsp_supply, memory_region, glink_edge, interconnects, interrupts_extended, smem_states, qmp`

## XBL / Boot Policy Evidence

| field | value |
| --- | --- |
| stock_retail_images | ABL, SPSS, ImageFv, FULL_ADSP, FULL_ADSP_DTB |
| patched_retail_images | ABL, SPSS, ImageFv, FULL_ADSP, FULL_ADSP_DTB, FULL_CDSP, FULL_CDSP_DTB |
| stock_sha256 | 94c07b0e4c71af8f9b5a5492e7b242ba9607a65b692189d17051ebc2e00c5102 |
| auth_verdict | hash_covered_no_go |


- Exact Odin NSP rule appears in `11` donor/firehose/XBL-like files.
- Direct bypass status: `CANDIDATE_PRECISE_BUT_HASH_COVERED`
- Direct bypass offset: `0x10935c`

## QSPA Init Policy

| field | value |
| --- | --- |
| imports_npu_policy | True |
| imports_nsp_policy | True |
| nsp_disabled_overrides_cdsprpcd | True |
| cdsprpcd_stop_property_gate | True |


## Policy Carriers

| name | source | verdict | terms |
| --- | --- | --- | --- |
| xuanyuan_images_OS3.0.303.0.WOACNXM_16.0__images__vendor_boot.img | Xiaomi 15 Ultra / xuanyuan | DIRECT_TARGET_PRESENT_INSPECT_ONLY | feature_id:1, qfprom:59, nsp:348, cdsp:644, qspa:2, rpmb:11 |
| IMAGES__devcfg.mbn | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | qfprom:1, nsp:1, rpmb:7 |
| IMAGES__devcfg.mbn | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | qfprom:1, nsp:1, rpmb:7 |
| IMAGES__dtbo.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | swfuse:6, cdsp:54 |
| IMAGES__dtbo.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | swfuse:6, cdsp:54 |
| IMAGES__featenabler.mbn | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | featureenabler:4, feature_id:15, configure_swfuse:8, swfuse:15, rpmb:41 |
| IMAGES__featenabler.mbn | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | featureenabler:4, feature_id:15, configure_swfuse:8, swfuse:15, rpmb:41 |
| IMAGES__vendor_boot.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | feature_id:1, qfprom:57, nsp:344, cdsp:645, qspa:2, rpmb:3 |
| IMAGES__vendor_boot.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | feature_id:1, qfprom:57, nsp:344, cdsp:645, qspa:2, rpmb:3 |
| RADIO__devcfg.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | qfprom:1, nsp:1, rpmb:7 |
| RADIO__devcfg.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | qfprom:1, nsp:1, rpmb:7 |
| RADIO__dtbo.img | Realme GT 7 Pro / RMX5011 | POLICY_CARRIER_CANDIDATE_INSPECT_ONLY | swfuse:6, cdsp:54 |


## Classification

- Label: `BOOT_POLICY_TRACE_BUILT`
- Best-fit disable path: `BOOTLOADER_OR_BOOT_POLICY_PATCHES_DT_STATUS_AND_QSPA_BOOTCONFIG`
- Reasoning:
- Android sees qspa.nsp=disabled and nsp=0xff before app/QNN routes matter.
- Active DT contains a complete-looking qcom,sun-cdsp-pas node but status=no.
- Kernel source exposes CDSP remoteproc/FastRPC support but no direct NSP CDSP gate was found.
- xbl_config excludes CDSP from RetailImages and CDSP-enabling changes are hash-covered.
