# Aime Active DT CDSP Map

Generated: 2026-06-29T14:32:01

Acceptance: `ACTIVE_DT_CDSP_NODE_MAPPED`, `CDSP_DEPENDENCY_CHAIN_MAPPED`

## Active Node

- Node: `/sys/firmware/devicetree/base/soc/remoteproc-cdsp@32300000`
- Status: `no`
- Compatible: `qcom,sun-cdsp-pas`
- Firmware names: `cdsp.mdt, cdsp_dtb.mdt`
- Properties: `clock-names, clocks, compatible, cx-supply, cx-uV-uA, firmware-name, glink-edge, interconnect-names, interconnects, interrupt-names, interrupts-extended, memory-region, mx-supply, mx-uV-uA, name, nsp-supply, nsp-uV-uA, phandle, qcom,qmp, qcom,smem-state-names, qcom,smem-states, reg, reg-names, status`

## Blocking / Missing Dependencies

| dependency | state | effect |
| --- | --- | --- |
| remoteproc status | no | prevents qcom_q6v5_pas CDSP probe from normal binding |


## Source Chain References

| category | file | line | text |
| --- | --- | --- | --- |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 117 | const char *firmware_name; |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 118 | const char *dtb_firmware_name; |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 161 | const char *dtb_firmware_name; |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 664 | ret = request_firmware(&adsp->dtb_firmware, adsp->dtb_firmware_name, adsp->dev); |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 666 | dev_err(adsp->dev, "request_firmware failed for %s: %d\n", |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 667 | adsp->dtb_firmware_name, ret); |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 671 | ret = qcom_mdt_pas_init(adsp->dev, adsp->dtb_firmware, adsp->dtb_firmware_name, |
| remoteproc_pas | drivers__remoteproc__qcom_q6v5_pas.c | 677 | ret = qcom_mdt_load_no_init(adsp->dev, adsp->dtb_firmware, adsp->dtb_firmware_name, |
| scm_pas | drivers__firmware__qcom_scm.c | 214 | ret = __scm_smc_call(NULL, &desc, probed_convention, &res, QCOM_SCM_CALL_ATOMIC); |
| scm_pas | drivers__firmware__qcom_scm.c | 232 | ret = __scm_smc_call(NULL, &desc, probed_convention, &res, QCOM_SCM_CALL_ATOMIC); |
| scm_pas | drivers__firmware__qcom_scm.c | 251 | * qcom_scm_call() - Invoke a syscall in the secure world |
| scm_pas | drivers__firmware__qcom_scm.c | 259 | static int qcom_scm_call(struct device *dev, const struct qcom_scm_desc *desc, |
| scm_pas | drivers__firmware__qcom_scm.c | 266 | return scm_smc_call(dev, desc, res, QCOM_SCM_CALL_NORMAL); |
| scm_pas | drivers__firmware__qcom_scm.c | 276 | * qcom_scm_call_atomic() - atomic variation of qcom_scm_call() |
| scm_pas | drivers__firmware__qcom_scm.c | 284 | static int qcom_scm_call_atomic(struct device *dev, |
| scm_pas | drivers__firmware__qcom_scm.c | 291 | return scm_smc_call(dev, desc, res, QCOM_SCM_CALL_ATOMIC); |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 37 | static int fastrpc_rpmsg_probe(struct rpmsg_device *rpdev) |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 54 | for (i = 0; i <= CDSP_DOMAIN_ID; i++) { |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 80 | err = fastrpc_init_privileged_gids(rdev, "qcom,fastrpc-gids", &data->gidlist); |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 134 | err = fastrpc_device_register(rdev, data, secure_dsp, domains[domain_id]); |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 139 | case CDSP_DOMAIN_ID: |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 142 | err = fastrpc_device_register(rdev, data, true, domains[domain_id]); |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 146 | err = fastrpc_device_register(rdev, data, false, domains[domain_id]); |
| fastrpc_rpmsg | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc_rpmsg.c | 291 | { .compatible = "qcom,fastrpc" }, |
| fastrpc_core | drivers__misc__fastrpc.c | 29 | #define CDSP_DOMAIN_ID (3) |
| fastrpc_core | drivers__misc__fastrpc.c | 2229 | static int fastrpc_device_register(struct device *dev, struct fastrpc_channel_ctx *cctx, |
| fastrpc_core | drivers__misc__fastrpc.c | 2248 | err = misc_register(&fdev->miscdev); |
| fastrpc_core | drivers__misc__fastrpc.c | 2274 | for (i = 0; i <= CDSP_DOMAIN_ID; i++) { |
| fastrpc_core | drivers__misc__fastrpc.c | 2317 | err = fastrpc_device_register(rdev, data, secure_dsp, domains[domain_id]); |
| fastrpc_core | drivers__misc__fastrpc.c | 2321 | case CDSP_DOMAIN_ID: |
| fastrpc_core | drivers__misc__fastrpc.c | 2324 | err = fastrpc_device_register(rdev, data, true, domains[domain_id]); |
| fastrpc_core | drivers__misc__fastrpc.c | 2328 | err = fastrpc_device_register(rdev, data, false, domains[domain_id]); |
| fastrpc_vendor_core | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc.c | 2140 | fl->cctx->domain_id == CDSP_DOMAIN_ID && |
| fastrpc_vendor_core | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc.c | 3259 | if (fl->cctx->domain_id == CDSP_DOMAIN_ID) { |
| fastrpc_vendor_core | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc.c | 3873 | if ((fl->cctx->domain_id != CDSP_DOMAIN_ID) \|\| (fl->pd_type != USERPD && |
| fastrpc_vendor_core | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc.c | 6029 | int fastrpc_device_register(struct device *dev, struct fastrpc_channel_ctx *cctx, |
| fastrpc_vendor_core | vendor__qcom__opensource__dsp-kernel__dsp__fastrpc.c | 6049 | err = misc_register(&fdev->miscdev); |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 45 | static int cdsp_loader_do(struct platform_device *pdev) |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 64 | "qcom,proc-img-to-load", |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 88 | priv->pil_h = rproc_get_by_phandle(rproc_phandle); |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 94 | dev_dbg(&pdev->dev, "%s: calling rproc_boot on %s\n", |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 96 | rc = rproc_boot(priv->pil_h); |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 98 | dev_err(&pdev->dev, "%s: rproc_boot failed with error %d\n", |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 137 | pr_debug("%s: going to call cdsp_loader_do\n", __func__); |
| cdsp_loader | vendor__qcom__opensource__dsp-kernel__dsp__cdsp-loader.c | 138 | cdsp_loader_do(cdsp_private); |
| socinfo | drivers__soc__qcom__socinfo.c | 288 | u32 num_subset_parts; |
| socinfo | drivers__soc__qcom__socinfo.c | 289 | u32 nsubset_parts_array_offset; |
| socinfo | drivers__soc__qcom__socinfo.c | 327 | socinfo_get_subpart_info(part_enum, part_info, num_parts); \ |
| socinfo | drivers__soc__qcom__socinfo.c | 721 | static uint32_t socinfo_get_num_subset_parts(void) |
| socinfo | drivers__soc__qcom__socinfo.c | 725 | le32_to_cpu(socinfo->num_subset_parts) : 0) |
| socinfo | drivers__soc__qcom__socinfo.c | 729 | static uint32_t socinfo_get_nsubset_parts_array_offset(void) |
| socinfo | drivers__soc__qcom__socinfo.c | 733 | le32_to_cpu(socinfo->nsubset_parts_array_offset) : 0) |
| socinfo | drivers__soc__qcom__socinfo.c | 738 | socinfo_get_subset_parts(void) |
| socinfo_header | include__soc__qcom__socinfo.h | 93 | SOCINFO_PART_NSP, |
| socinfo_header | include__soc__qcom__socinfo.h | 131 | int socinfo_get_subpart_info(enum subset_part_type part, |
| socinfo_header | include__soc__qcom__socinfo.h | 181 | int socinfo_get_subpart_info(enum subset_part_type part, |
| socinfo_header | include__soc__qcom__socinfo.h | 93 | SOCINFO_PART_NSP, |
| socinfo_header | include__soc__qcom__socinfo.h | 131 | int socinfo_get_subpart_info(enum subset_part_type part, |
| socinfo_header | include__soc__qcom__socinfo.h | 181 | int socinfo_get_subpart_info(enum subset_part_type part, |
| socinfo_header | include__soc__qcom__socinfo.h | 74 | SOCINFO_PART_NSP, |
| socinfo_header | include__soc__qcom__socinfo.h | 117 | int socinfo_get_subpart_info(enum subset_part_type part, |


## Classification

- Labels: `ACTIVE_DT_CDSP_NODE_MAPPED, CDSP_DEPENDENCY_CHAIN_MAPPED, NSP_CONSUMER_NOT_FOUND`
- First blocker: `active DT status=no on qcom,sun-cdsp-pas`
- Concrete NSP CDSP gate hits: `0`
- Socinfo NSP interface hits: `12`
