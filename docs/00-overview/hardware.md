# Hardware: The Odin 3 and the CQ8725S SoC

## The Device

The **AYN Odin 3** is a handheld Android gaming device. It ships with a Qualcomm SoC that, on paper, belongs to the SM8750 "sun" platform — the same silicon generation as Snapdragon 8 Elite. That shared silicon heritage is the source of a reasonable expectation: SM8750 has a Hexagon NPU. Shouldn't the Odin 3 have one too?

The answer is no — and the reason is in the chip part number.

---

## The SoC: CQ8725S-3-AA

The SoC in this device is not a Snapdragon 8 Elite. Its part number is **CQ8725S-3-AA** (`chip_id: CQ8725S`, `soc_id: 706`, revision `2.0`). The `CQ` prefix places it in Qualcomm's **Dragonwing** product line — the IoT and industrial segment, not the mobile flagship segment.

Live identity, as read from `/sys/devices/soc0` on the device:

```
soc_id:      706
revision:    2.0
chip_id:     CQ8725S
hw_platform: HDK
image_variant: CQ8725SA
```

The engineering build string on this unit is:

```
10:AQ3A.250728.001:eng.Odin3.20260616.191954
```

The `eng` prefix confirms this is an engineering/debug image, not a retail signed build. The bootloader firmware reports as `CQ8750-BOOT-R01-V1.1`; ADSP firmware is `CQ8750-ADSP-R01-V1.6`.

---

## What "Dragonwing / IoT Bin" Means

Qualcomm manufactures silicon at scale and then **bins** it: individual dies are tested, and depending on which functional blocks pass testing and which are fused off at the factory, a die gets assigned to a product SKU.

The same underlying silicon design can become:

| SKU family | Segment | Typical NPU state |
|---|---|---|
| Snapdragon 8 Elite (SM8750) | Mobile flagship | NSP enabled |
| CQ8725S (Dragonwing) | IoT / industrial | NSP fused OFF |

Binning is an economic and reliability practice. A die whose NPU block failed yield testing — or a die intentionally configured for a lower-capability market segment — gets its NSP fuse group blown during wafer-level or package-level test. That fuse cannot be unblown. The die ships as a "partial-goods" unit under a different SKU at a different price point.

This is not unique to Qualcomm. The practice is described in detail in Qualcomm patent **US 12,061,855** ("Functional circuit block harvesting"), which specifically covers disabling failed/binned blocks via OTP fuses read at boot. The kernel's `subset_parts` value (renamed from `defective_parts` in earlier platforms) is populated from QFPROM at boot and reflects exactly this.

On this device: `subset_parts = 9040 (0x2350)`, `num_subset_parts = 20 (0x14)`.

---

## The Socinfo Feature Byte Map

The SoC identity subsystem (`/sys/devices/soc0`) exposes per-feature presence bytes. A value of `0x0` means the feature is **present**; `0xff` means **absent**.

From the live device:

| Feature | Socinfo value | Meaning |
|---|---|---|
| `nsp` | `0xff` | **Absent — fused off** |
| `npu` | `0x0` | Present (legacy label) |
| `modem` | `0xff` | Absent — no cellular radio |
| `gpu` | `0x0` | Present |
| `video` | `0x0` | Present |
| `camera` | `0x0` | Present |
| `audio` | `0x0` | Present |
| `wlan` | `0x0` | Present |
| `display` | `0x0` | Present |

Two features are `0xff`. The modem absence is expected and easy to understand — the Odin 3 is a handheld gaming device with no cellular radio, and this is a genuine hardware bin-out reflected in the silicon. The `nsp=0xff` entry is the one that matters for this investigation.

> **Note** The `npu` field showing `0x0` is a legacy register label from an earlier platform generation. It does not refer to the Hexagon NPU/NSP in the SM8750 sense. The `nsp` field is the authoritative register for the neural processing subsystem on this platform.

---

## The "HDK" Platform

The `hw_platform` value `HDK` (Hardware Development Kit) indicates this SoC variant was characterized for development board / ODM-partner use — further consistent with a Dragonwing IoT-segment device rather than a consumer mobile flagship.

---

## What This Means for the Odin 3

AYN sourced the CQ8725S because it fits the Odin 3's positioning: a powerful, cost-optimized, gaming-focused SoC with strong GPU and media capabilities but without the premium modem or NPU that drive up cost in the mobile flagship tier. The NSP being fused off is a deliberate product segmentation decision made at Qualcomm's factory — it is upstream of AYN, and nothing AYN can do in firmware or software changes it.

See [`01-the-verdict/`](../01-the-verdict/index.md) for the complete causal chain.
