# VENDOR_DLKM_FAILURE_ROOT_CAUSE — Curated Evidence Copy

Project: Aime NPU Unlocker
Device: Odin 3 / `<SERIAL>` / Qualcomm `sun` / SM8750 (CQ8725S-3-AA)
Date: 2026-06-30
Author: offline postmortem (no live retry performed)

## CLASSIFICATION

**Primary cause: `AVB_METADATA_BROKEN`**
(dm-verity root-digest mismatch against the SIGNED vbmeta_a hashtree descriptor for `vendor_dlkm`.)

**Secondary independent defect: `EXT4_LAYOUT_BAD`**
(candidate `qcom_q6v5_pas.ko` inode is missing its `security.selinux` xattr — fatal to `insmod` under enforcing SELinux, but NOT the cause of the observed brick because the verity wall is hit first.)

**Exonerated:** `MODULE_SIGNATURE_OR_VERMAGIC_BAD`, `MODULE_DEP_METADATA_BAD` — ruled out with byte-level proof.

## ONE-LINE ROOT CAUSE

The candidate changed `vendor_dlkm` file content → the dm-verity Merkle root changed from stock `d0f1abeb…` to `d6572a40…` → but `vendor_dlkm`'s expected root digest is pinned inside the **SHA256_RSA4096-signed `vbmeta_a`** (which the unlocked bootloader still hands to the kernel) → at runtime `/vendor_dlkm` is mounted through a **dm-verity** device (`dm-18` = `vendor_dlkm-verity`) in **`veritymode=eio`** → every block read fails verification → the partition cannot mount → first-stage init dies before userspace → device became fully invisible (no ADB, no fastboot) and required a full stock `super` restore.

## EVIDENCE (proven, byte-level + live read-only)

### What actually flashed (timeline resolved)
There were two flash episodes on 2026-06-30:
1. `VENDOR_DLKM_A_PAS_ONEBOOT_PROBE` (09:33Z) — fastbootd only **resized**, never sent payload (`Resizing OKAY`, no `Sending`/`Writing`). Verdict `CANDIDATE_FLASH_INCOMPLETE_NO_PAYLOAD`. **Harmless** — device returned unchanged.
2. `candidate_oneboot_fastbootd` (11:33Z) — real write occurred: `Resizing OKAY / Sending 'vendor_dlkm_a' (145580 KB) OKAY / Writing OKAY`. `candidate_payload_written=true`. **This is the flash that broke boot.** Device then failed to return to ADB, went fully invisible, needed manual recovery → full `super` restore.

The flashed image was the **REHASHED** candidate:
- raw sha256 `e7a872c2bad5b5b363c4c5f6c97693692d65a4e5961d9610dcf38cf275f8a307`, size 149073920
- sparse round-trips exactly to that raw (`SPARSE_ROUNDTRIP_MATCH`), so sparse encoding was not corrupt
- pre-write device verified stock (`MODULE_SHA=d3b3b40f…`, extents reconstruct to stock `3cd4b65a…`)

### AVB / verity layer (PRIMARY — the smoking gun)
From signed stock `vbmeta_a` (EDL backup `edl-bootcritical-backup-20260628/images/vbmeta_a.img`):
- vbmeta Algorithm = **SHA256_RSA4096**, Authentication Block = 576 bytes (signed), pubkey sha1 `2597c218aae470a130f61162feaae70afd97f011`.
- Contains a **Hashtree descriptor for `vendor_dlkm`** (NOT a chain partition) directly in vbmeta_a's aux block:
  - Salt `b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b`
  - **Root Digest `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791`**
  - AVB flags = 0 → HASHTREE_DISABLED **unset**, VERIFICATION_DISABLED **unset** → verity fully enforced.
- Candidate `vendor_dlkm` AVB footer self-recomputed to root digest `d6572a40…` (internally consistent in its own footer) — **but the partition footer is not what the kernel trusts; vbmeta_a is.** Mismatch is guaranteed.

Live runtime confirmation (read-only `/proc/mounts` + sysfs dm names):
```
/dev/block/dm-18 /vendor_dlkm ext4 ro,seclabel,relatime,discard 0 0
dm-18 name=vendor_dlkm-verity     <-- /vendor_dlkm mounts THROUGH dm-verity
dm-6  name=vendor_dlkm_a           <-- underlying partition content
mapper: vendor_dlkm-verity -> dm-18 ; vendor_dlkm_a -> dm-6
ro.boot.veritymode = eio
```
Every sibling read-only partition is verity-wrapped identically (system-verity dm-14, vendor-verity dm-17, product-verity dm-16, etc.), confirming this is the platform's enforced AVB chain, not an artifact.

→ A changed `vendor_dlkm` digest under an enforced, signed verity root = block-layer mount failure = exactly the observed early-boot brick. **This is the dominant cause.**

### ext4 layer (SECONDARY independent defect — real, but downstream)
Carved ext4 fs (first 146636800 bytes) compared stock vs candidate:
- Filesystem **UUID identical** (`54c30959-3009-514a-8f4c-149a8c87e047`).
- Feature flags **byte-identical** (no journal, no metadata_csum, no fs-verity — expected for vendor_dlkm). Only `s_free_blocks_count` (108→101) and `s_wtime` changed — both expected from the write tool.
- `e2fsck -fn` on candidate: **clean, all 5 passes.**
- Module file `qcom_q6v5_pas.ko`: stock 131,832 B (33 blocks) → candidate 159,840 B (40 blocks, +28 KB). 7 extra blocks allocated at 35,692–35,698. FS has 35,800 blocks / 101 free → **no overflow.**
- Byte diffs fully localized: 6 metadata blocks (superblock, BG desc, group-1 block bitmap, group-1 inode table for inode 212) + the module's 40 data blocks. **Zero diffs in any other metadata.**
- **THE EXT4 DEFECT:** candidate inode 212 (`qcom_q6v5_pas.ko`) has its inline `security.selinux` xattr **zeroed out**. Stock = `u:object_r:vendor_file:s0`; candidate = EA magic+value all zero (`debugfs ea_get` → "key not found"). It is the **only** inode of 304 regular files missing this xattr.
  - Impact: on enforcing-SELinux Android, the module loader denies `insmod` of an unlabeled (or wrongly labeled) `.ko` → CDSP/PAS module would fail to load.
  - **But this wall is never reached** in the actual failure: dm-verity rejects the whole partition before any inode is read. It is a latent second bug in the candidate-build process, not the brick cause.

### module binary (EXONERATED)
Extracted `qcom_q6v5_pas.ko` from both images:
- **vermagic identical:** `6.6.87-android15-8-maybe-dirty-4k SMP preempt mod_unload modversions aarch64` (incl. the `-maybe-dirty` suffix). No mismatch.
- **Neither signed** — no `~Module signature appended~` trailer in stock or candidate; device does not enforce `module.sig_enforce` (stock vendor modules load unsigned). Not a rejection cause.
- **modversions CRCs:** all 120 shared `__versions` CRCs bit-for-bit identical; probe adds 4 valid kernel-export symbols (`strstr`, `param_ops_bool`, `of_device_is_compatible`, `of_find_node_opts_by_path`) consistent with added logging. Zero CRC mismatch.
- `modules.dep` / `modules.load` / `modules.alias` / `modules.softdep`: **byte-for-byte unchanged.** Only the `.ko` binary was swapped.

→ The module would have loaded fine on its own. The build problem is in the **image construction / rehash + xattr-preservation layer**, not the module.

## WHY FULL `super` RESTORE (not vendor_dlkm-only) WAS THE FIX
A full stock `super` restore rewrote all 18 sparse chunks, returning `vendor_dlkm_a` (dm-6) content to stock → verity root digest back to `d0f1abeb…` → matches signed vbmeta_a → mount succeeds → boot OK (with the expected orange/unlocked warning). A `vendor_dlkm`-only restore should also have sufficed in principle; the prior "vendor_dlkm-only not enough" note is attributable to the unstable fastboot link at the time (`Write to device failed (no link)`), not to AVB scope. vbmeta was never modified, so it did not need restoring.

## CONSEQUENCE FOR FUTURE WORK (hard rule)
**Any modified `vendor_dlkm` (or system/vendor/product/odm/system_dlkm/system_ext) flashed while the signed `vbmeta_a` is in place will brick boot the same way**, because all of those are verity-wrapped against digests pinned in the signed vbmeta. To boot ANY modified read-only partition you must FIRST defeat the verity enforcement — i.e. flash a vbmeta with HASHTREE_DISABLED set (`fastboot --disable-verity --disable-verification flash vbmeta …`). On an unlocked/orange device the bootloader does not re-verify the vbmeta signature, so a flag-flipped vbmeta is honored. (Feasibility of this, and whether it actually helps reach CDSP, is analyzed separately in the aime-npu-feasibility-20260630 artifact set.)

Additionally, ANY future modified vendor partition build MUST preserve `security.selinux` xattrs per-file (use `e2fsdroid`/`make_ext4fs` with the file_contexts, or edit in place via `debugfs set_inode_field`/`ea_set`, rather than a raw rebuild that drops xattrs). The xattr drop seen here would be a second, separate boot/insmod failure even after verity is handled.

## STATUS
`ROOT_CAUSE_PROVEN_NO_LIVE_RETRY_NEEDED`
- Primary: `AVB_METADATA_BROKEN` (signed-vbmeta verity root-digest mismatch).
- Secondary: `EXT4_LAYOUT_BAD` (missing SELinux xattr on the modified module).
- Module layer clean. Do not repeat this candidate. Do not flash any modified verity-backed partition without first disabling verity via vbmeta flags.
