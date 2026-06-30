# 07 · vendor\_dlkm & vendor\_boot One-Boot Probes

> **Status:** Probe approach fully designed, partially executed. The `vendor_dlkm`
> candidate flash on 2026-06-30T11:33Z bricked the device. See
> [Method 08](08-brick-and-recovery.md) for the full brick story.
> The `vendor_boot` probe was prepared but never executed.

---

## The Hypothesis

The kernel build ([Method 06](06-kernel-build.md)) produced log-only probe patches
for `qcom_q6v5_pas.ko`, `socinfo.ko`, `frpc-adsprpc.ko`, and `cdsp-loader.ko`
with vermagic and CRCs matching the stock kernel. These modules need to run on the
device to generate any meaningful log output.

The question: how do you get a modified `.ko` onto the device without rebuilding the
entire boot image stack?

The answer the probe attempted: replace the `.ko` inside `vendor_dlkm_a` — the
partition that hosts vendor dynamic kernel modules — boot it once to capture logs,
then restore the stock partition before Android fully settles.

This is the "one-boot probe" pattern. Write candidate → reboot → capture → restore.
Designed to be self-healing. In practice, it was not.

---

## Phase 0: Super One-Boot — Blocked on Slot B

The first route investigated was writing directly to `vendor_dlkm_a` via its
underlying super partition block device (`/dev/block/sda15`) from a running
Android userspace. This would avoid fastbootd entirely.

The super one-boot probe was mapped and preflight-passed — backups taken,
block layout confirmed, rollback path drafted — but hit a hard gate:

```
slot-unbootable:b = yes
slot-successful:b = yes
slot-retry-count:b = 6
```

Slot B was marked unbootable. The raw-super write path depends on slot B as a
hardware rollback: if the candidate breaks Android, the bootloader can retry from
slot B. With slot B unbootable, there is no reliable automatic fallback. The probe
was held at `BLOCKED_NO_SLOT_B_ROLLBACK_CONFIDENCE` and the fastbootd path was
pursued instead.

> Evidence: `evidence/aime-vendor-dlkm-oneboot-probe-20260630/super_oneboot_probe_20260630/SUPER_ONEBOOT_ROLLBACK_GATE_VERDICT.md`

---

## Phase 1: Fastbootd Route — Preflight

The fastbootd `vendor_dlkm` flash route requires the device to be in userspace
fastboot (fastbootd), not bootloader fastboot. The initial attempt failed because
Windows had no driver for the fastbootd USB product ID (`USB\VID_18D1&PID_4EE0`).

After installing the official Google Android USB driver (INF `oem84.inf`, SDK sha256
`cd1ebe134e3ace2b0c20b16ce62ca8369b1c80f497d6fa584f38dfbfc0a65ade`), the preflight
gate passed:

```
fastbootd is-userspace: yes
current-slot: a
has-slot:vendor_dlkm: yes
partition-size:vendor_dlkm_a: 0x8E2B000  (149,073,920 bytes)
```

> Evidence: `evidence/aime-vendor-dlkm-oneboot-probe-20260630/VENDOR_DLKM_DESTRUCTIVE_CHECKPOINT_READY.md`

---

## Phase 1a: First Flash Attempt — No Payload (Harmless)

**Timestamp:** 2026-06-30T09:33Z

The first actual flash run (`VENDOR_DLKM_A_PAS_ONEBOOT_PROBE`) resulted in:

```
fastboot flash vendor_dlkm_a <candidate>
Resizing 'vendor_dlkm_a' OKAY
[no Sending / no Writing]
```

The partition was resized but no payload was transferred. Fastboot reported
`CANDIDATE_FLASH_INCOMPLETE_NO_PAYLOAD_SENT`. Android returned normally after
the reboot. The device was unchanged — stock `qcom_q6v5_pas.ko` SHA-256
`d3b3b40f…` confirmed.

This was caused by a sparse encoding issue with the candidate image that prevented
fastbootd from accepting the payload. The candidate was re-encoded.

---

## The Candidate Image

The modified `vendor_dlkm_a` was built as follows:

- Start from stock `vendor_dlkm_a` raw image (sha256 `3cd4b65a…`, 149,073,920 B,
  mounted as a clean ext4 filesystem, UUID `54c30959-3009-514a-8f4c-149a8c87e047`)
- Replace `qcom_q6v5_pas.ko` (stock: 131,832 B / 33 ext4 blocks) with the patched
  version (159,840 B / 40 ext4 blocks, +28 KB of log-only probe instrumentation)
- Rehash the partition footer (AVB footer self-recomputed)
- Re-encode as sparse

Final candidate: sha256 `e7a872c2bad5b5b363c4c5f6c97693692d65a4e5961d9610dcf38cf275f8a307`,
size 149,073,920 B. Sparse round-trip verified (`SPARSE_ROUNDTRIP_MATCH`).

The ext4 filesystem was clean (`e2fsck -fn` passed all 5 passes). Block arithmetic
was sound: 35,800 blocks / 101 free after the replacement — no overflow.
The module's vermagic, CRCs, and dependency metadata were byte-identical to stock.

What the build process missed: the inode for the replaced `.ko` (inode 212) had its
`security.selinux` xattr zeroed out. Stock xattr: `u:object_r:vendor_file:s0`.
Candidate: EA magic + value all zero. This is a secondary defect that would have
prevented the module from loading under SELinux enforcing mode — but it was never
the cause of the brick. See [Method 08](08-brick-and-recovery.md).

---

## Phase 2: Second Flash Attempt — The Brick

**Timestamp:** 2026-06-30T11:33Z

The re-encoded candidate was flashed:

```
fastboot -s <SERIAL> --slot a flash vendor_dlkm \
  <artifacts>/vendor_dlkm_a_pas_probe_REHASHED_NO_FLASH.img

Resizing 'vendor_dlkm_a' OKAY
Sending 'vendor_dlkm_a' (145580 KB) OKAY
Writing 'vendor_dlkm_a' OKAY
```

`CANDIDATE_PAYLOAD_WRITTEN = true`. The device rebooted and did not return to ADB.
It did not return to fastboot. It disappeared from USB entirely.

> For the full forensics of what went wrong and why, see
> [Method 08 — Brick & Recovery](08-brick-and-recovery.md).

---

## vendor\_boot One-Boot Probe: Never Executed

In parallel with the `vendor_dlkm` approach, a `vendor_boot` one-boot probe was
designed. The `vendor_boot` partition on SM8750 devices carries QSPA init policy
and the first-stage ramdisk, both of which reference NSP/CDSP policy terms.

The theory: a modified `vendor_boot` could potentially change the `qspa.nsp`
property before Android read it, or force the CDSP DT node `status` from `no` to
`okay` in the first-stage ramdisk's copy of the DTB overlay.

The probe was prepared up to dry-run stage:

```
DRY RUN READY — rerun with -ExecuteFlash -IUnderstandVendorBootAFlash to flash vendor_boot_a
```

It was never executed. After the `vendor_dlkm` brick and recovery, all destructive
partition flashes were placed behind a new checkpoint requiring explicit approval
and verified verity handling. The `vendor_boot` probe shares the same verity
constraint: it too is covered by a hashtree digest pinned in the signed `vbmeta_a`.

> Evidence: `evidence/aime-vendorboot-oneboot-probe-20260630/VENDOR_BOOT_A_ONEBOOT_PROBE_RESULT.md`

---

## What the Probe Would Have Told Us

Had the `vendor_dlkm` probe survived boot, the expected log captures were:

| Outcome | Meaning |
|---------|---------|
| `QCOM_Q6V5_PAS_PROBE_PARAM_VISIBLE` | Patched module loaded; continue log analysis |
| `FASTRPC_CDSP_APPEARED` | CDSP came up; stop kernel work, move to QNN validation |
| `PROBE_BOOT_ADB_DID_NOT_RETURN_RESTORE_REQUIRED` | Restore path (what actually happened) |

The probe would not have enabled the NPU. It would have confirmed at what point
in the driver stack the failure occurs — whether at DT status evaluation, at the
SCM PIL authentication call, or deeper in the CDSP firmware load. That information
was lost when the partition mount failed before any userspace code ran.

---

## Timeline Summary

| Time (UTC) | Event | Verdict |
|------------|-------|---------|
| 2026-06-30T09:33Z | First flash attempt | `CANDIDATE_FLASH_INCOMPLETE_NO_PAYLOAD` — harmless |
| 2026-06-30T11:33Z | Second flash attempt | `CANDIDATE_PAYLOAD_WRITTEN` → boot failure → device invisible |
| 2026-06-30 (after) | Full stock super restore | Device recovered, stock confirmed |

> **Evidence files:**
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/VENDOR_DLKM_A_PAS_ONEBOOT_PROBE_RESULT.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/VENDOR_DLKM_DESTRUCTIVE_CHECKPOINT_READY.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/candidate_oneboot_fastbootd_20260630/CANDIDATE_VENDOR_DLKM_ONEBOOT_RESULT.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/candidate_oneboot_fastbootd_20260630/CANDIDATE_BOOT_TIMEOUT_RECOVERY_STATUS.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/super_oneboot_probe_20260630/SUPER_ONEBOOT_ROLLBACK_GATE_VERDICT.md`
