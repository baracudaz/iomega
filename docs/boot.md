# Installing Debian on an Iomega StorCenter ix2-dl

Verified working end-to-end on real hardware: fresh Debian 12 (bookworm) installed to the
internal SATA disk (software RAID1, degraded/single-disk), booted via the stock EMC U-Boot
1.1.4 — no U-Boot replacement, no NAND kernel/initrd partitions touched.

## Overview

1. Boot the official Debian bookworm armel/kirkwood netboot installer (`vmlinuz-6.1.0-50-marvell`
   + `initrd.gz`) from a FAT32 USB stick, via U-Boot's `bootm`.
2. Use the *real* Debian Installer's guided flow (network, mirror, `partman`) rather than manual
   `fdisk`/`e2fsprogs` — it already knows how to detect the SATA controller and set up software
   RAID.
3. Partition layout (MBR, not GPT):
   - `sda1` — 500MB, **plain ext2, NOT RAID** (U-Boot can't read through RAID metadata), mounted
     `/boot`.
   - `sda5` — 20GB, "physical volume for RAID" → degraded RAID1 (`md0`) → ext4, mounted `/`.
   - `sda6` — remainder (~980GB), same pattern → degraded RAID1 (`md1`) → ext4, mounted
     `/srv/data`.
   - When creating each RAID1 array in `partman`'s "Configure software RAID", the UI insists you
     "must select exactly 2" devices — selecting only the 1 available device is accepted anyway,
     producing a working degraded array. Add the second physical disk later with `mdadm --add`.
4. Decline the installer's own bootloader step (flash-kernel doesn't support this board) — the
   boot chain below is set up by hand afterward, from the installer's own rescue shell.
5. Install bodhi's community-maintained Kirkwood kernel (`linux-image-7.1.9-kirkwood-tld-1`, from
   [forum.doozan.com](https://forum.doozan.com/read.php?2,12096)) via `dpkg -i` in a chroot, then
   `apt install initramfs-tools` (pulls in a real `mdadm` binary into the initrd — critical, see
   Gotchas below) and `update-initramfs -c -k <version>`.
6. Build `uImage`/`uInitrd` and place them on `/boot` (the plain `sda1`), then set U-Boot's
   `bootcmd_sata` to load and boot them.

## Create images

```bash
cat vmlinuz-7.1.9-kirkwood-tld-1 kirkwood-lenovo-ix2-dl.dtb > zImage.fdt
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 \
  -n 'Debian-7.1.9-kirkwood-tld-1' -d zImage.fdt uImage
mkimage -A arm -O linux -T ramdisk -C gzip -a 0 -e 0 \
  -n initramfs -d initrd.img-7.1.9-kirkwood-tld-1 uInitrd
```

Copy both onto `/boot` (i.e. the root of `sda1`, not inside a `boot/` subdirectory — U-Boot loads
`/uImage` and `/uInitrd` directly off that partition).

## Boot - Final (verified working)

```bash
setenv bootcmd_sata 'ide reset; ext2load ide 1:1 0x00800000 /uImage; ext2load ide 1:1 0x03000000 /uInitrd; setenv bootargs console=ttyS0,115200 root=UUID=<root-fs-uuid> rootdelay=10; bootm 0x00800000 0x03000000'
run bootcmd_sata
```

Once confirmed working (reaches a login prompt), persist it:

```bash
setenv bootcmd 'run bootcmd_sata'
saveenv
```

Find `<root-fs-uuid>` from within the installed system with `blkid /dev/md0` (or whichever `mdX`
device holds `/`) — `root=UUID=...` is more robust than a device name, which can shift.

## Gotchas (the two bugs that cost the most time)

1. **The SATA disk is IDE device `1`, not `0`, on this U-Boot.** `ide part 0` reports "IDE device
   0 not available"; the real disk shows up under `ide part 1`. All prior draft commands in this
   file's history used `ide 0:1`, which silently no-ops (no error, just doesn't load anything) —
   very confusing to debug, since `bootm` then runs on whatever stale data happens to already be
   in RAM from a previous manual load.

2. **The kernel and ramdisk load addresses must be far apart.** This stock EMC U-Boot fork does
   not safely relocate/manage memory during `bootm` — if the ramdisk's load address is too close
   above the kernel's (e.g. `0x00800000` kernel / `0x01A00000` ramdisk, only ~18MB apart), the
   kernel silently overwrites the ramdisk in RAM while decompressing/expanding itself. U-Boot's own
   image checksum verification happens *before* this and reports "OK" for both images, so the
   corruption is invisible until the kernel panics moments later with:
   ```
   Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
   ...
   panic from mount_root_generic
   mount_root_generic from prepare_namespace
   ```
   This is the kernel's own fallback path when the initramfs was never actually unpacked — it
   happens even with `rdinit=/bin/sh` and no `root=` at all, which is the tell that the initrd
   itself never made it. The fix (per
   [forum.doozan.com](https://forum.doozan.com/read.php?2,70294,70549#msg-70549), same root cause
   documented for the whole EMC U-Boot NAS family — ix2-dl/ix4-200d/px2-300d) is simply to load the
   ramdisk much further out, e.g. `0x03000000` (giving the kernel ~40MB of headroom to expand into
   before hitting it). If a given kernel is even bigger, push this further still rather than assume
   the mechanism itself is broken.

3. Minor: U-Boot's own quoting only respects **single quotes** for grouping a multi-word `setenv`
   value with embedded semicolons — double quotes are not treated as a grouping mechanism, and each
   `;`-separated fragment gets executed immediately as a separate top-level command instead of being
   stored. (This can accidentally *run* the boot sequence right there at the prompt instead of just
   setting the variable — harmless, but confusing.)

## Base Setup

- [forum.doozan.com](https://forum.doozan.com/read.php?2,12096)
- [forum.doozan.com](https://forum.doozan.com/read.php?2,35331)
- [https://minaret.biz/tips/ix2/index.html#introduction]

## Swap

- [Linuxize: Add Swap Space on Debian 10](https://linuxize.com/post/how-to-add-swap-space-on-debian-10/)
- [DigitalOcean: Add Swap Space on Debian 10](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-debian-10)
