# MEIZE / KAIDA D115: Installing LineageOS (GSI)

A working method for running a LineageOS GSI on the **MEIZE D115** tablet (MediaTek MT8788, TrustKernel TEE), with **encryption working**.

> ⚠️ **Read the "Why most guides fail" section first.** The usual GSI step (`fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img`) **will not work on this tablet**. It boots into "Can't load Android system" every time.

## Tested GSIs

| GSI | Android | Result |
|---|---|---|
| **MisterZtr LineageOS 23.2 VANILLA-EXT4** (2026-05-24) | 16 | ✅ **Recommended.** Boots, encrypted, smooth |
| Andy Yan LineageOS 21 TD `arm64_bvN` (2026-11-09) | 14 | ✅ Works (Wi-Fi, audio, touch, rotation, encryption) |
| Andy Yan LineageOS 24 "pure" `gsi_arm64` (2026-11-07) | 17 | ❌ Hard-resets during early boot (APEX mount). Likely kernel 4.19 |
| Google Android 17 GSI through DSU (locked bootloader) | 17 | ❌ Won't boot |

Ready-made `vbmeta_system` images for the working GSIs are on the [Releases](../../releases) page.

---

## Device facts

| | |
|---|---|
| SoC | MT8788 (`mt6771` platform), 4×A73 + 4×A53, Mali-G72 MP3 |
| Bootloader | MediaTek `lk`, product `tb8788p1_64_bsp_k419` |
| Stock | Android 14 (`UP1A.231005.007`), vendor Android 12 (VNDK 31) |
| Kernel | 4.19 → **use ext4 GSIs only** (EROFS needs 5.4+) |
| Partitions | A/B, dynamic `super`, virtual A/B, `init_boot` |
| TEE | TrustKernel (TKCore), `keymaster@4.1-service.trustkernel` |
| `/metadata` | lives on the **`md_udc`** partition, not `metadata` |
| BROM | Not protected (no SLA/DAA). mtkclient works |

---

## Why most guides fail (the important part)

The TrustKernel keymaster gets a "root of trust" from the bootloader, based on **`vbmeta_a`**. If `vbmeta_a` is modified in **any** way (disabled flags, re-signed, even signed with the AOSP test key), the TEE logs:

```
ERR KEYMASTER:root_of_trust_verify: Invalid Root of Trust. Device might be compromised
```

Then keystore can't generate keys, `vold` fails `init_user0`, and the device drops to recovery with **"Can't load Android system"**. Turning encryption off in the vendor fstab gets you further, but `system_server` then crash-loops in `LockSettingsService` because it still needs keystore.

**Fix:** leave the stock `vbmeta_a` untouched. Re-sign only the **chained** `vbmeta_system_a` so it matches the GSI. On an unlocked device the bootloader tolerates the chained-key mismatch, and the TEE still trusts the boot.

---

## Requirements

- Linux PC with [platform-tools](https://developer.android.com/tools/releases/platform-tools) (fastboot 35+)
- [mtkclient](https://github.com/bkerler/mtkclient) for backups and unbricking
- `avbtool.py` and the AOSP test key `testkey_rsa2048.pem` (from `platform/external/avb`)
- An **ext4, arm64, A/B** GSI that ships with an AVB hashtree footer (Andy Yan's builds do)

## Step 0: Back up (do not skip)

Power off, hold **Vol+ and Vol−**, plug in USB (BROM mode):

```bash
python mtk.py r vbmeta_a,vbmeta_system_a,vbmeta_vendor_a,super,nvram,nvdata,persist,proinfo,seccfg \
  vbmeta_a.img,vbmeta_system_a.img,vbmeta_vendor_a.img,super.img,nvram.img,nvdata.img,persist.img,proinfo.img,seccfg.img
```

Keep `nvram`, `nvdata`, `persist`, `proinfo` and `seccfg` **private**. They contain your MAC addresses, serial number and keys.

## Step 1: Unlock the bootloader (wipes data)

Turn on **Developer options → OEM unlocking**, then:

```bash
adb reboot bootloader
fastboot flashing unlock      # confirm with Vol+ on the tablet
```

## Step 2: Build a vbmeta_system for your GSI

```bash
python3 avbtool.py info_image --image lineage.img     # must show a hashtree "Footer"
python3 avbtool.py make_vbmeta_image \
  --algorithm SHA256_RSA2048 --key testkey_rsa2048.pem --padding_size 4096 \
  --include_descriptors_from_image lineage.img \
  --output vbmeta_system_gsi.img
```

(If your GSI has no footer, add one: `avbtool.py add_hashtree_footer --image lineage.img --partition_name system --hash_algorithm sha256 --do_not_generate_fec`.)

## Step 3: Flash

```bash
# bootloader fastboot: flash the chained vbmeta only. Do NOT touch vbmeta_a
fastboot flash vbmeta_system_a vbmeta_system_gsi.img

# fastbootd: flash the GSI
fastboot reboot fastboot
fastboot delete-logical-partition product_a
fastboot flash system lineage.img

# bootloader fastboot: wipe (fastbootd can't see userdata on this device)
fastboot reboot bootloader
fastboot erase md_udc
fastboot -w
fastboot reboot
```

The first boot takes a few minutes. The bootloader shows an "orange state / unlocked" warning, which is normal.

## Updating or switching GSIs

Every new `system` image needs its own matching `vbmeta_system` (Step 2). If you flash a new `system` without one, it will fail dm-verity.

## Back to stock

Flash the stock images from the [Releases](../../releases) page:

```bash
fastboot flash vbmeta_a vbmeta_a.img
fastboot flash vbmeta_system_a vbmeta_system_a.img
fastboot flash super super.img          # xz -d first
fastboot erase md_udc && fastboot -w && fastboot reboot
```

Only relock (`fastboot flashing lock`) on a 100% stock image set.

## ⚠️ Known pitfalls after install

- **Don't use "Rooted debugging" / `adb root`.** On this tablet, restarting adbd as root crashes the USB gadget and can hang the device.
- **Avoid forced power-offs** (holding Power). On one test, a forced shutdown after the USB hang corrupted `/metadata` (the `md_udc` partition). Android reformatted it and the **file-encryption key was lost**, so the data could not be decrypted ("Can't load Android system", factory reset required). Clean restarts from the power menu are fine.
- Optional safety net: back up `md_udc` after setup (`python mtk.py r md_udc md_udc.img`). Writing it back restores access to your data, but only until the next factory reset.

## Changing the boot logo (the white "Meize" screen)

The first screen (white background with the Meize logo and "Powered by Android") lives in the **`logo`** partition, **not** in the bootloader. The partition is standard MediaTek format: a 512-byte header (`0x58881688`, name `logo`), then `count`, `total size`, an offset table, and **zlib-compressed raw BGRA** images. Two signing blocks (`cert1`, `cert2`) are appended, but an unlocked bootloader still accepts a modified logo.

- The Meize screen is **image index 42** (duplicate at **80**), stored **landscape 1280×800**. Indices 0 and 38 are unused MediaTek placeholders. The rest are charging animations.
- To replace it: decode, swap indices 42 and 80 with your own 1280×800 BGRA image, recompress, fix up the offsets and header size, then `fastboot flash logo logo_custom.bin` (bootloader fastboot, no data wipe).
- **Back up the original first** (`python mtk.py r logo logo_stock.bin`). To revert: `fastboot flash logo logo_stock.bin`.

## Debugging boot failures

MediaTek keeps kernel and TEE logs in the `expdb` partition, and they survive reboots. Read it with `python mtk.py r expdb expdb.img`, then
`strings -n 8 expdb.img | grep -E "Root of Trust|init_user0|FATAL|vold"`.

## Useful navigation

- Recovery menu: Vol− moves, Power selects. "Enter fastboot" = fastbootd, "Reboot to bootloader" = lk fastboot.
- BROM: power off fully (hold Power 20 s), then hold Vol+ and Vol− while plugging in USB.
