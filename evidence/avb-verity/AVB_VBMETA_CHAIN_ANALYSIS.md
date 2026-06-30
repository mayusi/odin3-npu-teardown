# AVB vbmeta_a Chain Analysis — vendor_dlkm Root Digest Anchor

**Date:** 2026-06-30  
**Subject:** Stock vbmeta_a (EDL backup) descriptor dump and vendor_dlkm digest anchor determination  
**Tool:** avbtool 1.3.0 (AOSP, pure Python)

---

## 1. Stock vbmeta_a — Full Descriptor Dump

**Source:** `edl-bootcritical-backup-20260628/images/vbmeta_a.img` (65536 bytes)  
**Command:** `python3 avbtool.py info_image --image vbmeta_a.img`

```
Minimum libavb version:   1.0
Header Block:             256 bytes
Authentication Block:     576 bytes
Auxiliary Block:          7168 bytes
Public key (sha1):        2597c218aae470a130f61162feaae70afd97f011
Algorithm:                SHA256_RSA4096
Rollback Index:           0
Flags:                    0
Rollback Index Location:  0
Release String:           'avbtool 1.3.0'
Descriptors:
    Chain Partition descriptor:
      Partition Name:          boot
      Rollback Index Location: 3
      Public key (sha1):       2597c218aae470a130f61162feaae70afd97f011
      Flags:                   0
    Chain Partition descriptor:
      Partition Name:          recovery
      Rollback Index Location: 1
      Public key (sha1):       2597c218aae470a130f61162feaae70afd97f011
      Flags:                   0
    Chain Partition descriptor:
      Partition Name:          vbmeta_system
      Rollback Index Location: 2
      Public key (sha1):       cdbb77177f731920bbe0a0f94f84d9038ae0617d
      Flags:                   0
    Prop: com.android.build.dtbo.fingerprint -> 'qti/sun/sun:15/AQ3A.250728.001/...'
    Prop: com.android.build.vendor_dlkm.os_version -> '15'
    Prop: com.android.build.vendor_dlkm.fingerprint -> 'qti/sun/sun:15/AQ3A.250728.001/...'
    Hash descriptor:
      Partition Name:        dtbo
      Digest:                2c5a4087ccd21f945075d77267e79c8488f6dc61653c20d5a6449c8a66942bbe
      Flags:                 0
    Hash descriptor:
      Partition Name:        init_boot
      Digest:                b54ddbfb494f904d7656d2fef9a6e2a9f53f3865f82d07c98ffe2e2f04918668
      Flags:                 0
    Hash descriptor:
      Partition Name:        vendor_boot
      Digest:                13f777ad480fe09df53ddd7b64acade99d21f204028bf857e2fde3153349e8cd
      Flags:                 0
    Hashtree descriptor:
      Partition Name:        odm
      Root Digest:           1522decad7ff017a19b0fe95a2083671e7d5628be4e7483bcc5b4a2ce7db9d35
      Salt:                  b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b
      Flags:                 0
    Hashtree descriptor:
      Partition Name:        system_dlkm
      Root Digest:           3c5bb40d6c7cf23d15e9fc0666f508ff0e203aa5bab16426a61641cd9cb5e9f2
      Salt:                  b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b
      Flags:                 0
    Hashtree descriptor:
      Partition Name:        vendor
      Image Size:            1471135744 bytes
      Root Digest:           c6ac4e5dc669e96e4ad63aea476dcf132b8370a093d1ccd1af87d55cae76557d
      Salt:                  b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b
      Flags:                 0
    Hashtree descriptor:                          <--- CRITICAL
      Version of dm-verity:  1
      Image Size:            146636800 bytes
      Tree Offset:           146636800
      Tree Size:             1163264 bytes
      Data Block Size:       4096 bytes
      Hash Block Size:       4096 bytes
      FEC num roots:         2
      FEC offset:            147800064
      FEC size:              1171456 bytes
      Hash Algorithm:        sha256
      Partition Name:        vendor_dlkm
      Salt:                  b43a0710416d68e58f79df0c2dfd40d12f6d3beed959f13d461c549e9ef83c8b
      Root Digest:           d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791
      Flags:                 0
```

---

## 2. Algorithm and Signature Status

- **Algorithm:** `SHA256_RSA4096` — vbmeta_a is **fully signed** with a 4096-bit RSA key.
- **Authentication Block:** 576 bytes (non-zero — signature is present and non-trivial).
- **Public key sha1:** `2597c218aae470a130f61162feaae70afd97f011`
- **Flags:** `0` — neither `HASHTREE_DISABLED` (bit 0) nor `VERIFICATION_DISABLED` (bit 1) is set.

---

## 3. Flags Analysis

| Flag | Bit | Value | Meaning |
|------|-----|-------|---------|
| HASHTREE_DISABLED | 0 | **0** | dm-verity hashtree enforcement is ON |
| VERIFICATION_DISABLED | 1 | **0** | AVB verification is ON |

Both disable flags are clear. dm-verity is active for all hashtree-described partitions.

---

## 4. Chain Structure for vendor_dlkm

vendor_dlkm is **NOT** chained via a Chain Partition descriptor. It is **directly embedded** as a `Hashtree descriptor` in `vbmeta_a` itself. The three Chain Partition descriptors in vbmeta_a point to: `boot`, `recovery`, and `vbmeta_system`. None chains vendor_dlkm. There is also no `vbmeta_vendor` image in the EDL backup (`edl-bootcritical-backup-20260628/images/` contains no vbmeta_vendor_a.img).

**Therefore:** the root digest for vendor_dlkm that the AVB chain trusts is the value **literally stored in the signed vbmeta_a binary:** `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791`.

---

## 5. Comparison of Root Digests

| Image | Root Digest | Salt |
|-------|-------------|------|
| Stock vbmeta_a (EDL backup, signed) | `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` | `b43a0710...` |
| Stock vendor_dlkm footer (AVB_INFO_STOCK.txt) | `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` | `b43a0710...` | 
| Candidate vendor_dlkm footer (AVB_INFO_REHASHED.txt) | `d6572a403c86314501c85dec874eafbe6a61374b7d4d30bffef515599918559e` | `b43a0710...` |

**Match between vbmeta_a and stock footer:** YES (identical digest and salt).  
**Match between vbmeta_a and candidate footer:** NO (digests differ at byte 2 onward: `d0f1...` vs `d6572a...`).

---

## 6. Crux: Which Root Digest Does the Kernel Use at Runtime?

This is the central question. The answer depends on how Android's init / first-stage init sets up the dm-verity table for vendor_dlkm.

### AVB two-path model

Android's AVB implementation (libavb + first-stage init) uses the following logic when setting up verity for a partition that has an AVB footer:

1. **The bootloader** reads `vbmeta_a` (the root of trust). Because the device is `orange` (unlocked), the **vbmeta_a signature is NOT verified by the bootloader** — but the bootloader still parses vbmeta_a and extracts the hashtree descriptors.

2. **The bootloader passes the full AVB result** (including all descriptor data from vbmeta_a, not from the partition footers) to the kernel via the kernel command line and/or the `androidboot.vbmeta.*` properties, specifically via the verity table parameters.

3. **First-stage init** (in ramdisk) sets up device-mapper verity (`dm-verity`) using the root digest from the **vbmeta_a descriptor**, NOT from the partition's own footer. The partition footer is used only if the partition is NOT described in any upstream vbmeta (i.e., it is "standalone" / "footer-only" mode). In this device's AVB layout, vendor_dlkm IS described in vbmeta_a, so first-stage init takes the hashtree parameters (root digest, salt, tree geometry) from vbmeta_a.

4. **Consequence:** The dm-verity table for vendor_dlkm will have root_digest = `d0f1abeb...` (from vbmeta_a). When the candidate vendor_dlkm partition is mounted, the kernel's dm-verity driver reads the partition data and recomputes the merkle tree hash. The computed root hash will be `d6572a40...` (matching the candidate's own footer), which does NOT equal the expected root hash `d0f1abeb...` from the verity table.

5. **With `veritymode=eio`:** a root-digest mismatch is NOT silently ignored and does NOT cause a kernel panic or reboot. Instead, every read to the mismatched blocks returns `EIO` (I/O error). However, the verity device is still set up and mounted. The EIO manifests the first time a read is attempted on any block that fails the hash check — which, for a root digest mismatch, means the ENTIRE partition returns EIO on first access (the root node of the Merkle tree fails, cascading to all leaves). This makes vendor_dlkm effectively unreadable, causing all kernel modules in vendor_dlkm to fail to load (`insmod` returns EIO), halting the boot at the point where critical modules (display, audio, storage controllers) are loaded.

### Note on orange state nuance

On `orange` (unlocked), the bootloader skips verifying the RSA signature on vbmeta_a, but it still reads and passes the descriptor data. The dm-verity root digest used by the kernel comes from what the bootloader parsed from vbmeta_a (passed through `dm-verity` setup), not from re-reading the partition footer. This is standard Android AVB architecture documented in AOSP `libavb` and `fs_mgr_avb`. The partition footer's own embedded vbmeta (algorithm NONE, no signature) is irrelevant at this layer; it is only consulted if avbctl or a standalone tool reads it directly. First-stage init uses the vbmeta chain result.

---

## 7. Verdict

**AVB/dm-verity root-digest mismatch is a LIKELY cause of the candidate boot failure.**

The signed stock `vbmeta_a` directly embeds a `Hashtree descriptor` for `vendor_dlkm` with root digest `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` (flags = 0, neither HASHTREE_DISABLED nor VERIFICATION_DISABLED is set). The candidate partition's recomputed root digest is `d6572a403c86314501c85dec874eafbe6a61374b7d4d30bffef515599918559e` — a definitive mismatch. Because first-stage init constructs the dm-verity table from the vbmeta_a descriptor (not the partition's own footer), the kernel enforces the `d0f1abeb...` expected digest regardless of the candidate's self-consistent footer. With `veritymode=eio`, all reads to vendor_dlkm return EIO, preventing any kernel modules from loading and causing boot failure. The self-consistency of the candidate's own AVB footer (algorithm NONE, flags 0) is irrelevant — it is never the authoritative source at runtime on this device's AVB layout.

**Confidence: HIGH.** The byte-level proof is that `vbmeta_a` (bytes parsed by avbtool at the Hashtree descriptor for vendor_dlkm in the Auxiliary Block) contains the literal hex string `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` under a SHA256_RSA4096-signed header with flags = 0.

---

## 8. Evidence Summary Table

| Item | Value |
|------|-------|
| vbmeta_a Algorithm | SHA256_RSA4096 (signed) |
| vbmeta_a Flags | 0 (HASHTREE_DISABLED=0, VERIFICATION_DISABLED=0) |
| vendor_dlkm descriptor type in vbmeta_a | Hashtree descriptor (direct, NOT chained) |
| Root Digest in vbmeta_a for vendor_dlkm | `d0f1abeb931505fb203c12059930e265c848b33fee174c6004276e1b32aa7791` |
| Root Digest in candidate partition footer | `d6572a403c86314501c85dec874eafbe6a61374b7d4d30bffef515599918559e` |
| Digests match? | **NO** |
| HASHTREE_DISABLED flag set? | **NO** |
| VERIFICATION_DISABLED flag set? | **NO** |
| veritymode on device | eio (EIO on mismatch, not panic/reboot) |
| ro.boot.verifiedbootstate | orange (unlocked — vbmeta_a sig not checked by BL, but digest still passed to kernel) |
| Verdict | AVB/verity root-digest mismatch is **LIKELY** the boot-break cause |

---

*Analysis performed 2026-06-30. avbtool 1.3.0 (AOSP). vbmeta_a source: EDL backup `edl-bootcritical-backup-20260628/images/vbmeta_a.img`.*
