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
   + `sda1` — 500MB, ext2, mounted `/boot`. Installed as a plain (non-RAID) partition, later
     converted to a degraded RAID1 (`md2`, `--metadata=1.0`) once a second disk was available —
     see "`/boot` on RAID1" below for why `metadata=1.0` specifically is required here.
   + `sda5` — 20GB, "physical volume for RAID" → degraded RAID1 (`md0`) → ext4, mounted `/`.
   + `sda6` — remainder (~980GB), same pattern → degraded RAID1 (`md1`) → ext4, mounted
     `/srv/data`.
   + When creating each RAID1 array in `partman`'s "Configure software RAID", the UI insists you
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

Find `<root-fs-uuid>` from within the installed system by checking which `mdX` device is mounted
at `/` (`mount | grep ' / '`), then `blkid` that device — `root=UUID=...` is more robust than a
device or `mdX` name, both of which can shift (see "md device renumbering" below — `/`'s `md`
number specifically is *not* stable across this project's own history).

## `/boot` on RAID1 (metadata=1.0)

`/` and `/srv/data` use mdadm's default metadata version (1.2), which is fine
because U-Boot never reads them directly — Linux mounts them after boot. `/boot` is different:
U-Boot's `ext2load` reads the raw partition (`ide 1:1`) directly, with zero awareness of RAID.
Default metadata versions don't work here:

+ **1.2** (mdadm's default) writes its superblock ~4KB into the start of the device — right where
  the ext2 boot sector/superblock lives. U-Boot would find RAID metadata instead of a filesystem.
+ **1.1** writes it at byte 0 — same problem, worse.
+ **1.0** writes it at the *end* of the device instead. The ext2 filesystem still starts at offset
  0, byte-for-byte identical to a non-RAID partition from U-Boot's point of view — it just ignores
  the trailing superblock it doesn't know exists. This is the same trick historically used for
  GRUB `/boot`-on-RAID before bootloaders understood mdadm natively.

Converting an existing plain `/boot` partition to this (done here on `sdb1`, degraded — no second
`/boot`-sized partition exists yet on the second disk):

```bash
cp -a /boot/. /root/boot-backup/        # back up first — recreate is destructive
umount /boot
mdadm --create /dev/md2 --level=1 --raid-devices=2 --metadata=1.0 /dev/sdb1 missing
mkfs.ext2 /dev/md2
mount /dev/md2 /boot
cp -a /root/boot-backup/. /boot/
# update /etc/fstab's /boot UUID to the new md2 filesystem's UUID (blkid /dev/md2)
mdadm --detail --scan /dev/md2 >> /etc/mdadm/mdadm.conf
update-initramfs -u                     # mdadm.conf changed, initrd.img gets regenerated...
mkimage -A arm -O linux -T ramdisk -C gzip -a 0 -e 0 \
  -n initramfs -d /boot/initrd.img-<version> /boot/uInitrd   # ...so uInitrd must be rebuilt too
```

Verified end-to-end with a real reboot: U-Boot's `ext2load` read the kernel/initrd off `sdb1`
exactly as before (it has no idea the partition is now a RAID member), and Linux auto-assembled
this array and mounted it at `/boot` normally afterward. Add the second member once the other disk
has a matching-size first partition: `mdadm --add /dev/sda1 /dev/md0` — **note the number**: this
array was `md2` when first created here, but is `md0` after the later renumbering below.

**Not yet verified: booting from the *other* disk.** `bootcmd_sata` hardcodes `ide 1:1` (Gotcha #1
below) — that happens to be whichever disk U-Boot enumerates as IDE device 1 with only one disk
attached. Once a real two-disk mirror exists, the whole point is surviving either disk failing —
so before trusting this setup, physically test booting with *only* the second disk connected (the
one currently at `sda`) and confirm U-Boot still finds it at the same `ide 1:1`, or update
`bootcmd_sata` if its bay position enumerates differently. Don't assume RAID1 membership alone
makes both disks equally bootable; only Linux-level (post-U-Boot) redundancy is proven so far.

## md device renumbering (`md0`=`/boot`, `md1`=`/`, `md2`=`/srv/data`)

The numbers above (`/`=`md0`, `/srv/data`=`md1`, `/boot`=`md2`) reflect the order things were
originally *created* in — the installer made root/data first, `/boot`'s RAID1 conversion came
later, hence `md2`. Purely for readability (numbers matching boot order, not creation order), this
was renumbered so `md0`=`/boot`, `md1`=`/`, `md2`=`/srv/data`. The array UUIDs, metadata versions,
and actual data are completely unaffected — an mdadm array's `/dev/mdN` number is just a label
assigned at assembly time, not something baked into the array itself. Nothing else on this box
references `/dev/mdN` directly: `/etc/fstab` mounts by filesystem UUID, and U-Boot reads `sdb1`'s
raw bytes directly (Gotcha #1 below) with zero awareness that Linux md exists at all — so this was
a config-only change with no risk to data or the boot chain:

```bash
# /etc/mdadm/mdadm.conf — edit each ARRAY line's /dev/mdN, matched by UUID (order doesn't
# matter since each line is independently identified by its own unique UUID):
#   old md0 (root, UUID c72bd2a3...)  -> new md1
#   old md1 (data, UUID 45abc5ce...)  -> new md2
#   old md2 (boot, UUID 7ed383f6...)  -> new md0
update-initramfs -u                     # picks up the renumbered mdadm.conf...
mkimage -A arm -O linux -T ramdisk -C gzip -a 0 -e 0 \
  -n initramfs -d /boot/initrd.img-<version> /boot/uInitrd   # ...so uInitrd must be rebuilt too
reboot
```

Verified with a real reboot: all three arrays reassembled under their new numbers automatically
(no live `mdadm --stop`/reassemble of the mounted root array ever needed — the new numbering only
takes effect at the next boot's early initramfs assembly), and every mount landed correctly
(`md0`→`/boot`, `md1`→`/`, `md2`→`/srv/data`). Each array's own `name=` field in `mdadm.conf`
(`debian:0`, `debian:1`, `iomega:2`) still reflects the old numbering — cosmetic only, harmless,
not worth chasing further.

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

## NAND flash — deliberately left untouched

This device has **no serial-recovery fallback**: `kwboot` doesn't work here because the serial
header is wired to UART1, not the UART0 the Kirkwood boot ROM listens to for recovery (per
[forum.doozan.com](https://forum.doozan.com/read.php?2,123034)). That makes the onboard U-Boot
binary and its environment the only thing standing between a working device and one that needs
disassembly to recover, so nothing here gets touched without a specific reason.

Stock NAND MTD layout (confirmed from `artifacts/logs/printenv.txt`, `nandconfig.txt`, `boot.txt`,
and both `dmesg-*.txt` — all agree exactly):

| Partition | Offset | Size | Status |
| --- | --- | --- | --- |
| `uboot` | 0x0 | 512KB | bootloader — never touched |
| `env` | 0xa0000 | 128KB | U-Boot environment — **actively used**, holds our `bootcmd`/`bootcmd_sata` |
| `env2` | 0xc0000 | 128KB | redundant/backup env |
| `uImage` | 0x100000 | 3MB | stock kernel (2.6.31.8) — untouched |
| `initrd` | 0x400000 | 4MB | stock initrd — untouched |
| `boot` | 0x800000 | ~1016MB | stock rootfs (UBI/UBIFS) — untouched |

Only `env`/`env2` are ever written (via `saveenv`). The kernel/initrd/rootfs partitions still hold
the original, unmodified stock Iomega/EMC firmware — this is deliberate, not an oversight: it means
the stock firmware is a ready emergency fallback if the SATA-disk boot chain above ever breaks,
reachable with no NAND reflashing at all.

**To revert to stock firmware** (e.g. `bootcmd_sata` stops working and needs debugging from a known
state), restore the original `bootcmd` and save:

```bash
setenv bootcmd 'run loadzimage; run loadinitrd; run mk_mtdparts; setenv bootargs $(console) $(bootargs_root) $(bootargs_end) $(mtdparts); bootm 0x40000 0x900000'
saveenv
```

(Exact value copied from `artifacts/logs/printenv.txt`, captured before any changes were made to
this device.) Then power-cycle — it boots the original Iomega firmware from NAND, unrelated to
whatever state the SATA disk or its `bootcmd_sata` is in.

**U-Boot binary backup:** `artifacts/firmware/uboot-nand-dump.bin` is a raw dump of the `uboot`
MTD partition (`mtd0`, 512KB), taken 2026-08-26 via `nanddump -f - /dev/mtd0`. Read twice and
compared (identical, 0 ECC failures, 0 bad blocks) before trusting the dump. Actual binary content
is ~475KB; the rest of the 512KB partition is erased (`0xFF`) padding. Be clear-eyed about this
backup's actual value given the no-`kwboot`-recovery situation above: the boot ROM reads this
partition *before* Linux ever runs, so this file can only help in the narrow case where NAND gets
corrupted while the box is still up and hasn't rebooted yet (`nandwrite` the good copy back before
rebooting) — it is **not** a brick-recovery tool for a device that's already failed to boot.

Same treatment for the environment partitions: `artifacts/firmware/uboot-env-dump.bin` (`mtd1`) and
`uboot-env2-dump.bin` (`mtd2`), 128KB each, same day, same double-read verification (identical, 0
ECC failures, 0 bad blocks — and `env`/`env2` legitimately differ from *each other*, which is
normal for U-Boot's redundant-environment scheme, not a read error). Confirmed both contain the
real, current environment (`strings` shows `bootcmd_sata` matching the live value exactly) — these
are actual usable backups, unlike the `uboot` binary above: if the environment ever gets corrupted
or a bad `saveenv` bricks the *variables* (not the bootloader binary itself), these can be
`nandwrite`-restored from a USB-booted rescue environment, which is a real, working recovery path
on this hardware (unlike `kwboot`).

Also backed up the stock kernel/initrd partitions: `uboot-kernel-dump.bin` (`mtd3`, 3MB) and
`uboot-initrd-dump.bin` (`mtd4`, 4MB) — same double-read verification, both clean (0 ECC failures,
0 bad blocks). Both confirmed as *valid* U-Boot legacy uImages via `file` (which validates the
header CRC to print details, not just magic-byte sniffing): kernel is `Linux-2.6.31.8` (matches
this doc's documented stock kernel version exactly) with intact CRCs, initrd's payload starts with
a genuine `BZh9` bzip2 magic. Both dated November 2014 — consistent with untouched original
factory firmware, never modified since this project began. These two are large enough (7MB
combined) that they're the closest thing to a full stock-firmware boot chain backup this repo
has, short of the `boot` UBI rootfs partition itself (~1016MB — not backed up, impractically large
for this repo and lowest-value to restore given it's never touched or at risk from this project's
own changes).

### Reading/writing the U-Boot environment from Linux (`fw_printenv`/`fw_setenv`)

No need to reach for the serial console for routine environment inspection or edits — `u-boot-tools`
(installed by the `base` Ansible role) provides real `fw_printenv`/`fw_setenv` binaries, properly
built for this system's actual libc (unlike bodhi's old static binpack tools intended for the
*stock* firmware's minimal userspace — see
[forum.doozan.com](https://forum.doozan.com/read.php?3,27280), which even shows a GLIBC-version
mismatch bug report for that exact scenario; not a concern here since this is just the normal
Debian-packaged tool).

The one thing that has to be right is `/etc/fw_env.config` — **do not use the whole-flash absolute
offsets from the MTD table above** (`env`@0xa0000, `env2`@0xc0000). This kernel already exposes
each NAND partition as its own MTD device node (`cat /proc/mtd`: `mtd1`=`uboot_env`,
`mtd2`=`uboot_env2`), so each one starts at offset 0 of *itself*, not of the whole chip. A config
using the absolute offsets against these partition-relative devices fails outright ("Cannot read
environment") — confirmed the hard way. The correct config (what the `base` role deploys):

```
# MTD device name	Device offset	Env. size	Flash sector size
/dev/mtd1	0x0	0x20000	0x20000
/dev/mtd2	0x0	0x20000	0x20000
```

Verified both directions: `fw_printenv` reads back `bootcmd`/`bootcmd_sata` byte-for-byte matching
what was set via serial, and a disposable `fw_setenv claude_test hello` / `fw_setenv claude_test`
round-trip was cross-checked against the real U-Boot prompt over serial before trusting it for
anything real — worth repeating that kind of check after any config change here, given this device
has no serial-recovery fallback if the environment ever gets corrupted.

Serial console is still needed for the one thing `fw_setenv` can't do: test-run an *unpersisted*
`bootcmd_sata` change (`run bootcmd_sata` at the prompt) before committing to it. `fw_setenv`
writes directly to NAND immediately — there's no "try before you save" step like U-Boot's own
`setenv` (RAM-only) → `saveenv` (persist) split. Use `fw_setenv` for variables you're already
confident in (or just reading current values); use serial for anything touching the boot chain
itself.

## Base Setup

+ [forum.doozan.com](https://forum.doozan.com/read.php?2,12096)
+ [forum.doozan.com](https://forum.doozan.com/read.php?2,35331)
+ <https://minaret.biz/tips/ix2/index.html#introduction>

## Swap

+ [Linuxize: Add Swap Space on Debian 10](https://linuxize.com/post/how-to-add-swap-space-on-debian-10/)
+ [DigitalOcean: Add Swap Space on Debian 10](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-debian-10)
