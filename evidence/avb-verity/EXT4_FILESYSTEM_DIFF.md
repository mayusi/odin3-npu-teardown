# EXT4 Filesystem Diff — vendor_dlkm boot-failure postmortem

**Date:** 2026-06-30  
**Analyst:** offline WSL forensics (dumpe2fs 1.47.0, debugfs, e2fsck, cmp)  
**Images:** stock `vendor_dlkm_a_stock_raw_20260630.img` vs candidate `vendor_dlkm_a_pas_probe_REHASHED_NO_FLASH.img`  
**Method:** Carved first 146 636 800 bytes (35 800 × 4 096-byte blocks) from each image, leaving AVB hashtree/FEC footer untouched.

---

## 1. Superblock Field Comparison

| Field | STOCK | CANDIDATE | Status |
|---|---|---|---|
| Filesystem UUID | `54c30959-3009-514a-8f4c-149a8c87e047` | `54c30959-3009-514a-8f4c-149a8c87e047` | **SAME** |
| Magic | `0xEF53` | `0xEF53` | OK |
| s_state (fs state) | `1` (clean) | `1` (clean) | OK |
| s_inodes_count | 352 | 352 | OK |
| s_blocks_count_lo | 35 800 | 35 800 | OK |
| s_free_blocks_count | 108 | 101 | DIFF (expected: +7 blocks for larger module) |
| s_free_inodes_count | 32 | 32 | OK |
| s_mnt_count | 0 | 0 | OK |
| s_wtime (last write) | `0x495c0780` (2009-01-01 placeholder) | `0x6a430615` (2026-06-29 23:56:05 UTC) | DIFF (expected: tool wrote image) |
| s_mtime (last mount) | 0 | 0 | OK |
| s_creator_os | 0 (Linux) | 0 (Linux) | OK |
| s_rev_level | 1 | 1 | OK |
| s_feature_compat | `0x00000028` | `0x00000028` | **SAME** |
| s_feature_incompat | `0x00000042` | `0x00000042` | **SAME** |
| s_feature_ro_compat | `0x0000407b` | `0x0000407b` | **SAME** |
| Default mount options | user_xattr acl | user_xattr acl | OK |
| Journal present? | NO (no `has_journal` in compat flags) | NO | OK |
| Checksum type | CRC32c (group descriptors only; no metadata_csum) | CRC32c | OK |

**Feature decode (identical in both):**
- `s_feature_compat`: `ext_attr, dir_index`
- `s_feature_incompat`: `filetype, extents`
- `s_feature_ro_compat`: `sparse_super, large_file, huge_file, gdt_csum, dir_nlink, extra_isize, shared_blocks`

No `verity` or `metadata_csum` flags in either image. No `readonly` flag. Feature sets are byte-for-byte identical.

---

## 2. Block Group Descriptor Comparison

| Field | Group 0 STOCK | Group 0 CAND | Group 1 STOCK | Group 1 CAND |
|---|---|---|---|---|
| Block bitmap | block 2 | block 2 | block 32770 | block 32770 |
| Inode bitmap | block 3 | block 3 | block 32771 | block 32771 |
| Inode table | blocks 4–14 | blocks 4–14 | blocks 32772–32782 | blocks 32772–32782 |
| Free blocks | 0 | 0 | 108 | 101 |
| Free inodes | 0 | 0 | 32 | 32 |
| csum | `0x6e69` | `0x6e69` | `0x1180` | `0x8f53` |

Group 1 checksum differs (`0x1180` → `0x8f53`) because the free-block count changed from 108 to 101. This is a valid, recalculated checksum — e2fsck confirms it clean.

---

## 3. e2fsck Results

```
STOCK:
e2fsck 1.47.0 (5-Feb-2023)
Pass 1-5: all clean
vendor_dlkm: 320/352 files (2.2% non-contiguous), 35692/35800 blocks

CANDIDATE:
e2fsck 1.47.0 (5-Feb-2023)
Pass 1-5: all clean
vendor_dlkm: 320/352 files (2.5% non-contiguous), 35699/35800 blocks
```

**e2fsck candidate: CLEAN — no filesystem errors detected.**

---

## 4. Module File Comparison — `qcom_q6v5_pas.ko`

| Attribute | STOCK | CANDIDATE |
|---|---|---|
| Inode | 212 | 212 |
| Mode | `0644` | `0644` |
| Size | **131 832 bytes** (128.7 KB) | **159 840 bytes** (156.1 KB) |
| Delta size | — | +28 008 bytes (+21.2%) |
| Blockcount (×512) | 264 → 33 × 4 K-blocks | 320 → 40 × 4 K-blocks |
| Extents | `(0-32): 33804–33836` | `(0-32): 33804–33836, (33-39): 35692–35698` |
| Timestamps (ctime/mtime/atime) | 2009-01-01 placeholder | 2026-06-29 23:56:05 UTC |
| Inode flags | `0x80000` (EXTENTS_FL) | `0x80000` (EXTENTS_FL) |
| i_file_acl_lo (external EA block) | 0 | 0 |
| **`security.selinux` xattr** | **`u:object_r:vendor_file:s0`** | **MISSING (zeroed out)** |

The candidate module is 7 four-kilobyte blocks larger than stock and allocated a second extent at blocks 35 692–35 698. Total filesystem is 35 800 blocks; 101 free blocks remain — **no capacity overflow**.

---

## 5. CRITICAL FINDING: Missing `security.selinux` xattr on candidate module

**Stock inode 212 inline EA (offset 128+32 = 160 of the 256-byte inode):**
```
00 00 02 ea  07 06 40 00  00 00 00 00  1a 00 00 00   <- EA magic + header
00 00 00 00  73 65 6c 69  6e 75 78 00  ...           <- "selinux"
value: u:object_r:vendor_file:s0
```

**Candidate inode 212 inline EA (same offset):**
```
00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00   <- ALL ZEROS — no EA magic
```

`debugfs ea_get` confirms: stock returns `u:object_r:vendor_file:s0`; candidate returns *"Extended attribute key not found"*.

**Scope:** Scanning all 352 inodes in both images:
- STOCK: 304 regular files, **all 304 have `security.selinux`**
- CANDIDATE: 304 regular files, **303 have it; inode 212 (`qcom_q6v5_pas.ko`) does not**

This is the **only** inode in the entire filesystem where the xattr was lost.

---

## 6. Byte-Diff Summary

```
Total differing bytes: 88 971  across 44 blocks
```

| Region | Blocks | Diff bytes | Class |
|---|---|---|---|
| Superblock (block 0) | 1 | 6 | METADATA — free_blocks count + wtime + CRC |
| BG descriptor (block 1) | 1 | 3 | METADATA — group 1 free_blocks + checksum |
| Group 1 block bitmap (block 32770) | 1 | 2 | METADATA — 7 newly-allocated blocks marked used |
| Group 1 inode table (block 32774) | 1 | 62 | METADATA — inode 212 updated (size, timestamps, extents, **EA cleared**) |
| Module data — extent 0 (blocks 33804–33836) | 33 | 79 292 | FILE DATA — modified .ko binary |
| Module data — extent 1 NEW (blocks 35692–35698) | 7 | 9 606 | FILE DATA — overflow bytes of larger module |

**No diffs in any other metadata block.** All superblock/descriptor/bitmap/inode-table changes are precisely accounted for by the single file replacement. The EA clearance is embedded inside the 62 bytes of inode-table change on block 32774.

---

## 7. Verdict

```
EXT4_LAYOUT_BAD
```

**The ext4 filesystem structure is internally self-consistent** (e2fsck clean, valid checksums, no corrupt blocks, UUID unchanged, feature flags unchanged, capacity not exceeded). However the candidate image contains a **filesystem-level defect that is fatal at runtime**:

> **Inode 212 (`/lib/modules/qcom_q6v5_pas.ko`) has its `security.selinux` xattr wiped to zeros.** The stock label `u:object_r:vendor_file:s0` is absent in the candidate. On an enforcing SELinux Android system, a kernel module with no SELinux label is either denied load by the kernel's `insmod` path or the module loader fails the security check, preventing CDSP/DSP subsystem bring-up and causing the device to stall/reboot before userspace comes up.

**Strongest single fact:** `debugfs ea_get security.selinux /lib/modules/qcom_q6v5_pas.ko` → stock returns label, candidate returns *"key not found"* — the xattr inline EA block in inode 212 is all zeros in the candidate image.

This is not an AVB/hashtree problem (the candidate was correctly re-hashed with matching root digest `d6572a...`). The failure mode is: image mounts fine from e2fsck's perspective, dm-verity passes (new hashtree), but SELinux denies the module at `insmod` time because the file has no label.

---

*Evidence files: carved FS images at `/dev/shm/stock_fs.img` and `/dev/shm/cand_fs.img` (WSL session-local; re-carved from originals in `aime-vendor-dlkm-acquire-20260630/`).*
