# MEIZE / KAIDA D115 – Stock Firmware Backup

Stock firmware dumped from a **MEIZE D115** tablet (manufacturer: KAIDA) with [mtkclient](https://github.com/bkerler/mtkclient).

| | |
|---|---|
| SoC | MediaTek MT8788 (MT6771 platform, Helio P60 family) |
| Android | 14 (build `UP1A.231005.007`), vendor Android 12 (VNDK 31) |
| Security patch | 2024-10-05 |
| Kernel | 4.19.191 |
| Partitions | A/B, dynamic (`super`), virtual A/B |
| Treble | Yes, full VNDK (GSI-capable), arm64 |

## Files (see Releases)

| File | Description |
|---|---|
| `super.img.xz` | Full `super` partition (system/vendor/product), slot `_a`, raw 8 GB, xz-compressed |
| `vbmeta_a.img` | Stock `vbmeta_a` |
| `SHA256SUMS` | Checksums |

Device-unique partitions (nvram, nvdata, persist, proinfo, seccfg) are **not** included. Back up your own before flashing anything.

## Restore to stock

```bash
xz -d super.img.xz
# with an unlocked bootloader (bootloader fastboot, not fastbootd):
fastboot flash super super.img
fastboot flash vbmeta_a vbmeta_a.img
fastboot reboot
# or with mtkclient in BROM mode:
python mtk.py w super,vbmeta_a super.img,vbmeta_a.img
```

Use at your own risk. Only for the MEIZE/KAIDA D115.
