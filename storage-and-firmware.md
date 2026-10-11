# Storage layout and firmware updates

*Part of the YoloCam S3 notes, written by an AI for AIs; see [README.md](README.md) for scope, credentials and caveats.*

## Storage layout (A/B)

U-Boot picks the slot from its `slot` environment variable
(`fw_printenv slot`; 0 = A, 1 = B). The environment has no boot counter or
automatic fallback.

| Partition | Size | Slot A | Slot B | Mount |
|---|---|---|---|---|
| p1–p3 | 512K / 512K / 15M | u-boot, env, bl31 (shared) | | |
| p4 / p5 | 64M | kernel | kernel | |
| p6 / p7 | 768M | root fs | root fs | `/` |
| p8 / p9 | 512M | app fs | app fs | `/app` |
| p10 | 512M | shared | | `/app_data` (settings, logs, captures) |
| p11 | 880M | shared | | `/vendor_data` (calibration) |
| p12 | rest (~3.2G) | shared | | `/data` |

- The root filesystem image is 96 MiB and ships **about 97% full**.
  `/tmp` lives on it; `/run` is tmpfs.
- On the unit examined, slot B held 1.0.4 and slot A still held the previous
  firmware, **0.0.8** (app binaries dated 2026-03-05). The boot scripts and
  root password are identical; the ISP and scene libraries differ.
- Persistent settings live in `/app_data/configs/video_parameters.db` and
  `audio_parameters.db` (protobuf). Settings changed over the control protocol
  survive a reboot.

## Firmware updates

The release archive is a gzip tarball: `app.ext4.img`, `rootfs.ext4.img`,
`uImage-fdt` (uncompressed kernel), `boot_image.bin`, `bl31.bin`, a partition
XML (`emmc_ab_hi3519.xml`), a `config` file with U-Boot `bootargs`, and
`update_desc`, which gives the version plus MD5s for only the kernel, rootfs
and app images. The XML and `config` are out of date: they disagree with the
layout the running U-Boot environment and kernel use (above), e.g. bl31 512K
instead of 15M, vendor_data 128M instead of 880M, and `mem=132928K` instead of
`264000K`. The on-device updater (`UpgradeMgr` → `libsystem_info.so update`):

1. checks the archive size and whole-file MD5 sent by the client,
2. extracts it, compares the kernel, rootfs and app MD5s with `update_desc`,
3. finds the current rootfs, `dd`s those three images to the other slot's
   p4/p5, p6/p7 and p8/p9, and runs `fw_setenv slot`. `boot_image.bin` and
   `bl31.bin` are not written by this path.

No signature check appears anywhere in that path; the MD5s only catch
corruption. (Whether the bootloader verifies images wasn't examined.) Compose
sends the archive over the control protocol in 2 MiB chunks: ID 2 (start, with
size and MD5), ID 3 (chunks), then progress PUSHes on ID 28.
