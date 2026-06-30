# 08 · The Brick and Recovery

> **This page documents a device-bricking event that occurred on 2026-06-30.**
> The Odin 3 went completely invisible — no ADB, no fastboot, no USB device —
> after a modified `vendor_dlkm_a` was flashed through fastbootd.
> Recovery required a full stock `super` partition restore.
> The cause has been forensically proven at the byte level.

---

> **Warning**
>
> The flash operation described on this page **bricked the device**. It is
> documented here as a cautionary record, not as a procedure to follow.
> The hard lesson learned is stated at the end of this page and must be read
> before any future modified-partition flash is attempted.

---

## What Was Flashed

**Partition:** `vendor_dlkm_a`
**Timestamp:** 2026-06-30T11:33:08Z
**Method:** fastbootd (userspace fastboot, `is-userspace: yes`)

The candidate image contained one change relative to stock: `qcom_q6v5_pas.ko`
was replaced with a log-instrumented build (+28 KB of `pr_info` calls, no
force-enable logic). The ext4 filesystem reconstructed cleanly. The module's
vermagic, CRCs, and `modules.dep` metadata were byte-for-bit identical to stock.
The partition was re-encoded as sparse and the AVB footer inside the image was
self-recomputed to reflect the new Merkle root.

The flash output:

```
Resizing 'vendor_dlkm_a'         OKAY
Sending 'vendor_dlkm_a' (145580 KB) OKAY
Writing 'vendor_dlkm_a'          OKAY
Finished.
```

`CANDIDATE_PAYLOAD_WRITTEN = true`. Candidate sha256:
`e7a872c2bad5b5b363c4c5f6c97693692d65a4e5961d9610dcf38cf275f8a307`.

The device then rebooted. It never came back.

---

## What Happened: The Device Went Dark

After the reboot following the flash:

- ADB: no device
- fastboot: no device
- `USB\VID_18D1&PID_4EE0` (fastbootd): not present
- Qualcomm/Android USB PnP device: not present
- The device display showed nothing

The device was physically power-cycled using hardware keys. It still did not
appear on USB. It entered a state of being completely invisible to the host PC —
no enumeration, no fallback mode, nothing. This persisted for an extended period.

---

## Root Cause: The Forensic Proof

The cause is precise and fully proven from byte-level artifacts and live device
reads taken before the flash. The full analysis is in
`evidence/vendor-dlkm-postmortem/VENDOR_DLKM_FAILURE_ROOT_CAUSE.md`.

### The Verity Chain

The Odin 3's stock `vbmeta_a` partition contains a **Hashtree descriptor** for
`vendor_dlkm` embedded directly in its authentication block (not a chain partition
— the descriptor is inline). This vbmeta is signed with **SHA256_RSA4096**, pubkey
sha1 `2597c218aae470a130f61162feaae70afd97f011`. Its contents:

| Field | Value |
|-------|-------|
| Algorithm | SHA256_RSA4096 |
| Authentication block | 576 bytes, signed |
| Hashtree descriptor for `vendor_dlkm` | present, inline in aux block |
| Salt | `b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b` |
| **Root digest (stock)** | `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` |
| HASHTREE_DISABLED flag | **unset** (verity enforced) |
| VERIFICATION_DISABLED flag | **unset** (verification enforced) |

The vbmeta_a was taken from the EDL backup made on 2026-06-28
(`evidence/edl-bootcritical-backup-20260628/images/vbmeta_a.img`). It was not
modified at any point. It was not flashed. It sat in `vbmeta_a` exactly as
AYN/Qualcomm shipped it.

### What the Candidate Did

Replacing `qcom_q6v5_pas.ko` changed the content of the `vendor_dlkm` ext4
filesystem. Different content = different Merkle tree = different dm-verity root
digest. The candidate's AVB footer self-computed to:

**Root digest (candidate):** `d6572a40…` (internally consistent in its own footer)

But the footer inside the flashed partition is not what the kernel trusts.
The kernel trusts `vbmeta_a`. And `vbmeta_a` says the correct root digest is
`d0f1abeb…`.

`d6572a40` ≠ `d0f1abeb`. The mismatch is guaranteed and irrecoverable from boot.

### How the Kernel Mounts vendor\_dlkm

Live `/proc/mounts` and dm mapper state read before the flash:

```
/dev/block/dm-18  /vendor_dlkm  ext4  ro,seclabel,relatime,discard  0  0

dm-18  name=vendor_dlkm-verity    ← /vendor_dlkm mounts THROUGH dm-verity
dm-6   name=vendor_dlkm_a         ← underlying partition content
```

And the critical property:

```
ro.boot.veritymode = eio
```

`veritymode=eio` is the strict enforcement mode. When the dm-verity Merkle tree
verification fails for any block in the partition, the block device returns EIO
(I/O error) to every read. This is not a soft warning. Every file read from
`/vendor_dlkm` fails at the block layer. The ext4 driver cannot mount the filesystem.
First-stage init cannot find the files it needs. Boot fails before userspace.

Every sibling read-only partition — `system` (dm-14), `vendor` (dm-17), `product`
(dm-16), `system_dlkm`, `odm` — is verity-wrapped identically. This is the
platform's full AVB enforcement chain, not an artifact of one partition.

### Why It Went Fully Invisible

The failure mode is not a soft boot loop that cycles back to fastboot. When
first-stage init fails this early — at `vendor_dlkm` mount, which happens before
userspace services start — the system enters a crash state that does not reliably
re-enter fastboot on its own. The bootloader had already handed off to the kernel;
the kernel crash did not trigger the hardware watchdog in a way that landed in
bootloader mode. The device's display showed nothing. USB was not enumerated.
No Android, no bootloader, no EDL.

The device was physically inaccessible to software until hardware-key recovery was
performed (power + volume key combination to force bootloader entry).

---

## Secondary Defect: SELinux xattr Drop (Would Have Been the Next Problem)

The candidate `qcom_q6v5_pas.ko` had a second defect independent of verity:
inode 212 in the ext4 filesystem had its `security.selinux` xattr zeroed out.

- **Stock:** `u:object_r:vendor_file:s0`
- **Candidate:** EA magic + value all zero (key not found)

On enforcing-SELinux Android, the module loader rejects `insmod` of an unlabeled
`.ko`. Had verity somehow been defeated and the partition mounted, the module
would have failed to load at insmod time.

This defect is documented, not as the brick cause, but as a lesson for any
future `vendor_dlkm` build process. It arose because the image reconstruction
tool dropped xattrs when repacking the ext4 image. The correct tooling is
`e2fsdroid` with the platform `file_contexts`, or editing in place via
`debugfs ea_set`.

---

## The Module Was Fine

For completeness: the `qcom_q6v5_pas.ko` binary itself was not the problem.

| Check | Stock | Candidate | Match |
|-------|-------|-----------|-------|
| vermagic | `6.6.87-android15-8-maybe-dirty-4k SMP preempt mod_unload modversions aarch64` | identical | Yes |
| Module signed | No | No | N/A |
| `__versions` CRCs (120 entries) | — | bit-for-bit identical | Yes |
| New symbols added (4) | — | `strstr`, `param_ops_bool`, `of_device_is_compatible`, `of_find_node_opts_by_path` — valid kernel exports | OK |
| `modules.dep` | stock | byte-for-byte unchanged | Yes |
| `modules.load` | stock | byte-for-byte unchanged | Yes |

The module would have loaded correctly on its own. The problem was in the
**image construction layer** — rehashing changed the Merkle root, and xattr
preservation was dropped.

---

## Recovery

### Attempt 1: vendor\_dlkm-only restore (inconclusive)

With the device recovered to bootloader via hardware keys, then switched to
fastbootd, a stock `vendor_dlkm_a` restore was attempted:

```
fastboot -s <SERIAL> --slot a flash vendor_dlkm \
  <artifacts>/vendor_dlkm_a_stock_raw_20260630.img

Resizing 'vendor_dlkm_a' OKAY
Sending 'vendor_dlkm_a' OKAY
Writing 'vendor_dlkm_a' OKAY
Finished
```

After `fastboot reboot`, the device still did not return to ADB within five
minutes. USB remained invisible. This was later attributed to an unstable
fastboot link at that moment (the subsequent session showed `Write to device
failed (no link)` errors indicating a marginal connection), not to an AVB
scope problem. The `vbmeta_a` was never modified and covers `vendor_dlkm`
only — restoring `vendor_dlkm_a` alone is *in principle* sufficient.

### Attempt 2: Full stock super restore (success)

A full stock `super` restore was performed — all 18 sparse chunks written,
returning the entire dynamic partition set (including `vendor_dlkm_a`) to
stock state. The stock raw image sha256 for `vendor_dlkm_a` was confirmed:

```
d3b3b40f64711d120264f75b2e512851ca109ba1954a1eb70ca61e221a3f88c9  (qcom_q6v5_pas.ko)
3cd4b65a25cb317a7c3479b8c65eb5fc751a728c95e6be45beb44274f80f6eb8  (vendor_dlkm_a raw)
```

After the full super restore, the device returned to Android with the unlocked
bootloader warning (orange state — expected). Confirmed live state:

```
boot_completed     = 1
slot               = _a
verified_boot_state = orange
ro.boot.veritymode = eio
/vendor_dlkm       = mounted read-only from dm-18
qcom_q6v5_pas.ko   SHA-256 = d3b3b40f…  (stock confirmed)
NSP                = 0xff
QSPA NSP           = disabled
```

The device was fully recovered with no permanent damage.

---

## The Hard Lesson

> **Warning**
>
> **Any modified `vendor_dlkm`, `system`, `vendor`, `product`, `odm`,
> `system_dlkm`, or `system_ext` partition flashed to this device while the
> signed `vbmeta_a` is in place will brick boot in exactly the same way.**
>
> All of these partitions are covered by hashtree descriptors pinned inside
> the SHA256_RSA4096-signed `vbmeta_a`. Changing any file in any of these
> partitions changes the Merkle root. The signed vbmeta does not update
> automatically. The mismatch causes block-layer EIO under `veritymode=eio`.
> First-stage init fails. The device goes dark.
>
> **Before flashing any modified read-only partition, you must disable verity
> enforcement first:**
>
> ```
> fastboot --disable-verity --disable-verification flash vbmeta <vbmeta.img>
> ```
>
> On an unlocked (orange) device, the bootloader does not re-verify the vbmeta
> signature, so a flag-flipped vbmeta is honored. This step must come *before*
> any modified partition is written — never after.
>
> Additionally, any modified partition build must preserve `security.selinux`
> xattrs on every file. Use `e2fsdroid` with the platform `file_contexts`, or
> edit xattrs in place via `debugfs ea_set` / `debugfs set_inode_field`. A raw
> ext4 rebuild that drops xattrs will produce a second, independent failure at
> SELinux `insmod` time, after verity is handled.

---

## Timeline

| Time | Event |
|------|-------|
| 2026-06-30T09:33Z | First flash — no payload sent, device unchanged |
| 2026-06-30T11:33Z | Second flash — payload written, device bricked |
| 2026-06-30 (after 11:33Z) | Hardware-key recovery to bootloader |
| 2026-06-30 | fastbootd reached; vendor_dlkm-only restore attempted (link unstable) |
| 2026-06-30 | Full stock super restore performed |
| 2026-06-30 | Android confirmed booted, all stock hashes verified |

> **Evidence files:**
> `evidence/vendor-dlkm-postmortem/VENDOR_DLKM_FAILURE_ROOT_CAUSE.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/candidate_oneboot_fastbootd_20260630/CANDIDATE_VENDOR_DLKM_ONEBOOT_RESULT.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/candidate_oneboot_fastbootd_20260630/CANDIDATE_BOOT_TIMEOUT_RECOVERY_STATUS.md`
> `evidence/aime-vendor-dlkm-oneboot-probe-20260630/post_recovery_android_alive_20260630/RECOVERY_AFTER_VENDOR_DLKM_FAILURE_VERDICT.md`
> `evidence/edl-bootcritical-backup-20260628/images/vbmeta_a.img` (the signed vbmeta that enforced verity)
