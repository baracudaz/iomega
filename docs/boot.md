# Installing Debian on an Iomega StorCenter ix2-dl

Verified working end-to-end on real hardware: fresh Debian 12 (bookworm) installed to the
internal SATA disk (software RAID1, degraded/single-disk), booted via the stock EMC U-Boot
1.1.4 — no U-Boot replacement, no NAND kernel/initrd partitions touched. **This NAND-untouched
state no longer holds** — see the 2026-09-11 update below; kept here only as an accurate
description of the original bring-up.

**Update (2026-08-27):** upgraded in place to Debian 13 (trixie) — see `ansible/README.md`'s "OS
release" section. Everything below describes the original bookworm bring-up and is still accurate
for the boot chain itself (unchanged by the OS upgrade); only the installed userspace release has
moved on.

**Update (2026-09-11):** the boot chain itself changed — `bootcmd` now boots straight from NAND
(`bootcmd_nand`) instead of the SATA disk's `/boot` partition (`bootcmd_sata`, still intact as
fallback). NAND's `kernel`/`initrd`/`boot` MTD partitions (not `uboot`/`env`/`env2`) were resized
and overwritten to make this possible — see "Booting from NAND" under "NAND flash" below for the
full story. Everything above (the SATA install itself, RAID setup, `mkimage` recipe) is still
accurate and still how the images loaded by `bootcmd_nand` are actually built; only *where U-Boot
loads them from* has changed.

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

**Verified 2026-09-11: booting with both disks attached.** `bootcmd_sata` hardcodes `ide 1:1`
(Gotcha #1 below), which was previously only tested with a single disk present. `md0`/`/boot` and
`md1`/`/` were mirrored onto a second, physically much larger disk (`sda`, 4TB, added via
`mdadm --add` after partitioning it to exactly replicate the original disk's MBR layout — see
"`/boot` on RAID1 (metadata=1.0)" above), then the box was rebooted with both disks attached.
Result: clean ~70s boot, `ide reset`/`ext2load ide 1:1` found `/boot` correctly, both arrays came
back `[2/2] [UU]`. This confirms U-Boot's IDE probe tolerates a second disk being present at boot
and doesn't get confused about which one is "device 1" — the previously open risk here.

**Still not verified: booting from *only* the second disk** (the original `sdb` physically
disconnected, `ide 1:1` then necessarily resolving to `sda`). The test above proves two-disk
enumeration is safe when the original disk is still device 1; it does not prove `sda` is
independently bootable if `sdb` is ever removed. Confirm that separately before relying on `sda`
as a true failover boot disk, not just a data mirror.

**Also confirmed by the same fork this session (not yet folded into this doc's main narrative):
this stock EMC U-Boot 1.1.4 cannot read GPT partition tables** — verified via a real
forum.doozan.com thread run on byte-for-byte identical firmware
([https://forum.doozan.com/read.php?2,70294](https://forum.doozan.com/read.php?2,70294)). This is
why the second disk above was wiped from its original GPT/exFAT layout and repartitioned as MBR
before being added to the arrays. A classic MBR partition table also hard-caps at ~2TiB regardless
of physical disk size (32-bit LBA sector count) — so even a 4TB drive only ever gets ~1.95TiB
actually usable under this boot chain, without a deeper redesign of how `/boot` is loaded.

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

## NAND flash

This device has **no serial-recovery fallback**: `kwboot` doesn't work here because the serial
header is wired to UART1, not the UART0 the Kirkwood boot ROM listens to for recovery (per
[forum.doozan.com](https://forum.doozan.com/read.php?2,123034)). That makes the onboard U-Boot
binary and its environment the only thing standing between a working device and one that needs
disassembly to recover, so `uboot`/`env`/`env2` never get touched without a specific reason, and
any write to `kernel`/`initrd`/`installer` (below) gets the same read-back-and-verify discipline.

**Historical stock layout** (confirmed from `artifacts/logs/printenv.txt`, `nandconfig.txt`,
`boot.txt`, and both `dmesg-*.txt` — all agreed exactly, as originally shipped):

| Partition | Offset | Size | Status (as shipped) |
| --- | --- | --- | --- |
| `uboot` | 0x0 | 512KB | bootloader |
| `env` | 0xa0000 | 128KB | U-Boot environment |
| `env2` | 0xc0000 | 128KB | redundant/backup env |
| `uImage` | 0x100000 | 3MB | stock kernel (2.6.31.8) |
| `initrd` | 0x400000 | 4MB | stock initrd |
| `boot` | 0x800000 | ~1016MB | stock rootfs (UBI/UBIFS) |

**Repurposed 2026-09-11**: decided the stock kernel/initrd/rootfs content had zero remaining value
once two 4TB SATA drives made the original small-disk stock firmware moot — see "Booting from NAND"
below for the full rationale, procedure, and current live layout. `uboot`/`env`/`env2` are
untouched throughout; everything from `kernel` onward was resized and overwritten.

**The "revert to stock firmware" instructions that used to live here no longer work — do not use
them.** They relied on `bootcmd_orig`'s `loadzimage`/`loadinitrd`, which read fixed-size chunks
from NAND using the U-Boot env vars `kernel_start`/`kernel_size`/`initrd_start`/`initrd_size`
(still `0x100000`/`0x300000`/`0x400000`/`0x400000` — U-Boot env is untouched by the DTB work
below). Those offsets now sit inside our own, differently-sized `kernel`/`initrd` content, so
`bootcmd_orig` would load truncated/wrong data, not the real stock firmware — it wouldn't brick
anything (worse case is just a boot failure, same recoverable-only-with-physical-access situation
as any other failed boot attempt here), but it's no longer a meaningful fallback. **The real
fallback now is `bootcmd_sata`** (still fully intact and independently tested — see "Booting from
NAND" below), not `bootcmd_orig`.

**U-Boot binary backup:** `artifacts/firmware/uboot-nand-dump.bin` is a raw dump of the `uboot`
MTD partition (`mtd0`, 512KB), taken 2026-08-26 via `nanddump -f - /dev/mtd0`. Read twice and
compared (identical, 0 ECC failures, 0 bad blocks) before trusting the dump. Actual binary content
is ~475KB; the rest of the 512KB partition is erased (`0xFF`) padding. Be clear-eyed about this
backup's actual value given the no-`kwboot`-recovery situation above: the boot ROM reads this
partition *before* Linux ever runs, so this file can only help in the narrow case where NAND gets
corrupted while the box is still up and hasn't rebooted yet (`nandwrite` the good copy back before
rebooting) — it is **not** a brick-recovery tool for a device that's already failed to boot.

Same treatment for the environment partitions: `artifacts/firmware/uboot-env-dump.bin` (`mtd1`) and
`uboot-env2-dump.bin` (`mtd2`), 128KB each, same double-read verification (identical, 0 ECC
failures, 0 bad blocks). These are actual usable backups, unlike the `uboot` binary above: if the
environment ever gets corrupted or a bad `saveenv` bricks the *variables* (not the bootloader
binary itself), these can be `nandwrite`-restored from a USB-booted rescue environment, a real,
working recovery path on this hardware (unlike `kwboot`). (These were captured before the
`bootcmd_nand` work below and no longer reflect the live environment content — re-dump if a
current env backup is ever needed; only the *mechanism* they document remains relevant.)

**How the redundant `env`/`env2` pair actually works** (verified empirically, not just from the
man page): each copy starts with a 4-byte CRC32 + a 1-byte generation counter, then the variable
data. `env` and `env2` are **not** meant to be identical — it's a "current + previous" pair, not a
mirror. Every `fw_setenv`/`saveenv` writes a fresh copy of the *entire* environment to whichever
side currently has the lower generation number, with counter = `max(both) + 1`; that side becomes
the new "current" one, and the other now correctly lags by exactly one save, as a deliberate
rollback point — not a fault. Confirmed by testing: writing a disposable variable correctly
alternated the target each time, and generation numbers tracked exactly as expected. One trap
worth knowing: `strings` isn't a safe way to inspect what's *actually* active in a dump — it
happily prints stale bytes sitting past the real double-NUL environment terminator, left over from
before a shorter value was last written there. Use `fw_printenv` (or a real parser) to check
what's actually live, not `strings`.

**Stock kernel/initrd/rootfs dumps (historical only, no longer live)**: `uboot-kernel-dump.bin`
(old `mtd3`, 3MB) and `uboot-initrd-dump.bin` (old `mtd4`, 4MB) in `artifacts/firmware/`, captured
2026-08-26 with the same double-read verification (both clean, 0 ECC failures/bad blocks),
confirmed as valid U-Boot legacy uImages via `file`: kernel `Linux-2.6.31.8`, initrd starting with
a genuine `BZh9` bzip2 magic, both dated November 2014. **These describe NAND state that no longer
exists** — `mtd3`/`mtd4` were overwritten 2026-09-11 (below) — kept purely as a historical record
of the original factory firmware. The old ~1016MB `boot` (UBI rootfs) partition was similarly
dumped in full before being overwritten — see "NAND archaeology" below for what was actually found
inside it, and why that dump mattered for timing (parts of it were physically destroyed by the
`kernel`/`initrd` rewrite below, since the new, bigger partitions' byte ranges overlap where the
old `boot` partition's data lived).

### NAND archaeology (what was actually in the stock `boot`/UBI partition, before it was overwritten)

Before repurposing anything, the full ~1016MB `boot` MTD partition was dumped (`nanddump`,
`/dev/mtd5` at the time) and analyzed offline (`ubireader`, `e2fsprogs`'s `debugfs`) — genuinely
useful context, not just curiosity, since it settled what (if anything) was worth preserving:

- It's **not a flat root filesystem** — the UBI volume holds a bundle of sub-images:
  `zImage`/`initrd` (u-boot legacy images), `apps` (682MB, ext2 — the real OS), `config` (8MB,
  ext2), `oem` (100KB, squashfs). Only ~384MB of the declared ~1016MB was actually written; the
  rest was already free space inside that UBI volume's own allocation.
- **This specific unit received a firmware update at some point**: the `zImage`/`initrd` bundled
  *inside* the `boot` volume are dated March/May 2012 and are byte-different (different sizes,
  different compression — gzip vs bzip2) from what was actually loaded via the live
  `kernel`/`initrd` MTD partitions (dated November 2014, matching the dumps above). The `boot`
  volume's bundle was the *original factory image*, preserved separately as a factory-reset
  payload, distinct from whatever was actually running.
- **The stock Iomega/EMC firmware's `apps` image is Debian underneath**: mounted read-only via
  `debugfs`, `/etc/os-release` → `PRETTY_NAME="Debian GNU/Linux 7 (wheezy)"`, `/etc/debian_version`
  → `7.1`. EMC's proprietary layer (`SDKApps` plugin framework, `/usr/local/appmd/appmd` — the
  actual binary behind the `executord` init script, their web-management backend) sat on top of a
  genuine Debian wheezy userspace and the stock 2.6.31.8 kernel — a fitting bit of irony for a
  project whose whole goal is installing mainline Debian on this hardware.
- **One concrete feature gap worth knowing about**: stock `/etc/init.d` included `iscsi-scst`/
  `scst` (iSCSI target support) — something this project's current ansible roles (`nfs`, `samba`)
  don't replicate. Not implemented, just worth knowing it existed if iSCSI LUN export is ever
  wanted. No custom `udev` rules existed (empty directory) and `executord` turned out to be
  Iomega's management-UI daemon, not a GPIO/LED/fan controller — so no low-level hardware-control
  logic was found worth porting; that's handled at the kernel-driver level, already covered by
  bodhi's `kirkwood-tld` kernel.
- The full original dump is kept locally at `artifacts/firmware/uboot-boot-dump.bin` (~1GB) —
  **gitignored, not committed** (too large; the other `*-dump.bin` files are small enough to
  commit, this one isn't). Local reference only.

### Booting from NAND (2026-09-11)

**Motivation**: `/boot` living on the SATA disk, MBR-partitioned, is what forces the ~2TiB ceiling
discussed above (U-Boot's `ext2load ide 1:1` needs an MBR-readable partition table on that disk).
Once NAND's `kernel`/`initrd` partitions hold the real boot images instead, U-Boot never needs to
read *any* SATA partition table again — clearing the way for the SATA disks to eventually become
full GPT and use their actual capacity. (Actually repartitioning the SATA disks as GPT is separate,
not-yet-done follow-on work.)

**Key discovery that made this safe to attempt incrementally**: the NAND MTD partition layout
(`kernel`/`initrd`/`boot` sizes above) is **not** stored on NAND itself, and **not** driven by a
U-Boot `mtdparts=` cmdline var either (`bootcmd_sata`'s `bootargs` never set one, yet `/proc/mtd`
still showed all 6 partitions correctly) — it's defined by the **device tree**
(`kirkwood-lenovo-ix2-dl.dtb`), **appended directly onto the kernel image itself**
(`CONFIG_ARM_APPENDED_DTB=y`) that's wrapped as `/boot/uImage` and loaded from the SATA disk.
Confirmed via `dmesg`'s `"6 fixed-partitions partitions found on MTD device orion_nand"` (the
Linux OF/device-tree MTD parser's log signature, not the cmdline parser's), a live
`/sys/firmware/devicetree/base/mbus@f1000000/nand@12f/` tree matching `/proc/mtd` exactly, and
`/proc/cmdline` containing no `mtdparts=` at all. This meant the new partition layout could be
built and tested **via the existing, proven SATA boot path, with zero NAND writes** — a bad
edit could at worst produce wrong `mtdN` devices in `dmesg`, since NAND partitioning is completely
decoupled from whether the kernel can mount the real (SATA) root filesystem.

**New layout** (`uboot`/`env`/`env2` byte-identical to before; the source DTB
— `/usr/lib/linux-image-7.1.9-kirkwood-tld-1/kirkwood-lenovo-ix2-dl.dtb`, confirmed byte-identical
to the live running tree — is kept in this repo alongside the edited version):

| Partition | Offset | Size | Purpose |
| --- | --- | --- | --- |
| `kernel` | 0x100000 | 16MB | ~3x the actual `uImage` (5.6MB) — headroom for future kernel updates |
| `initrd` | 0x1100000 | 32MB | ~3x the actual `uInitrd` (11.3MB) — headroom for initramfs growth |
| `installer` | 0x3100000 | ~975MB | **Reserved, not yet populated** — see below |

Intended for a Debian installer/rescue image, bootable straight from NAND independent of the SATA
disks entirely — upgrading physical-access recovery from "limited U-Boot prompt" to "a real
installer/rescue shell." Populating it is separate, not-yet-done follow-on work.

All offsets/sizes are eraseblock-aligned (128KB erase size: `16MB/128KB=128`, `32MB/128KB=256`,
`~975MB/128KB=7800`, all exact).

Procedure actually followed, staged in three parts (matching this project's general practice of
testing the smallest-blast-radius step first):

1. **Edit + test via SATA, zero NAND risk.** Decompiled the packaged DTB (`dtc -I dtb -O dts`),
   edited *only* the `nand@12f` partition child nodes (kernel/initrd resized, `boot` renamed to
   `installer` and shrunk to the remainder), recompiled, and diffed the round-tripped result
   against the original to confirm *only* the intended nodes changed. Rebuilt a test kernel image
   the same way as "Create images" above (`cat vmlinuz... newdtb.dtb > zImage.fdt; mkimage ...`),
   placed it as a second file on `/boot` (original `/uImage` untouched), pointed `bootcmd_sata` at
   the new filename only, and rebooted. Result: clean ~10s boot, `/proc/mtd` and the live device
   tree showed exactly the new 3-partition layout, nothing else regressed. `bootcmd_sata` was then
   reverted back to the original `/uImage` — this step only proved the layout, it didn't adopt it.
   DTB/DTS artifacts kept in `artifacts/firmware/`: `kirkwood-lenovo-ix2-dl.dtb`/`.dts` (original)
   and `kirkwood-lenovo-ix2-dl-resized.dtb`/`.dts` (edited, actually deployed below).

2. **Write real content into the now-larger NAND partitions** (still no `bootcmd` change — the box
   kept booting from SATA exactly as before regardless of this step's outcome, since nothing reads
   NAND `kernel`/`initrd` yet at this point):
   ```bash
   flash_erase /dev/mtd3 0 0 && nandwrite -p /dev/mtd3 uImage-resized   # the tested kernel+dtb image from step 1
   flash_erase /dev/mtd4 0 0 && nandwrite -p /dev/mtd4 /boot/uInitrd    # the real, active initrd
   ```
   Verified via full read-back (`nanddump` + `cmp` against the exact source byte lengths): **both
   partitions matched byte-for-byte, 0 bad blocks, 0 ECC corrections** on either write. Writing via
   the partition device nodes (not raw whole-flash offsets) is structurally bounded by the
   kernel's own MTD layer — an offset/size mistake errors out rather than silently overrunning
   into the neighboring, still-untouched `env`/`env2`/`uboot` partitions.

3. **Wire up and cut over to `bootcmd_nand`** — the first point where `bootcmd` itself changed
   what actually boots, so treated with the same rigor as the earlier disk-mirror reboot test
   (full pre-flight, then a real reboot, since there's no serial console for an unpersisted test):
   ```
   bootcmd_nand=nand read.e 0x00800000 0x100000 0x1000000; nand read.e 0x03000000 0x1100000 0x2000000; setenv bootargs console=ttyS0,115200 root=UUID=9c5fb1da-4a31-4ad5-8c25-0b3c2fbdf904 rootdelay=10; bootm 0x00800000 0x03000000
   ```
   Same RAM addresses as the proven `bootcmd_sata` (`0x00800000`/`0x03000000`, already satisfying
   the Gotcha #2 headroom requirement), same `bootargs`, `nand read.e <ram_addr> <nand_offset>
   <size>` in place of `ext2load`. No `mtdparts`/`mk_mtdparts` needed — this kernel gets its NAND
   layout from the device tree, not the cmdline parser (see the discovery above), so unlike
   `bootcmd_orig`/`bootcmd_usb` this doesn't need to construct or pass one. Set as a **new**
   variable — `bootcmd_sata` was left completely untouched as an immediate one-command fallback
   (`setenv bootcmd 'run bootcmd_sata'; saveenv`) if this needed backing out.

   **Verified working end-to-end with a real reboot**: `bootcmd` set to `run bootcmd_nand`, box
   rebooted, came back in ~10 seconds. Confirmed via `dmesg`: `Kernel command line` matched exactly
   what `bootcmd_nand` sets, `OF: fdt: Machine model: Lenovo IX2-dl` (the resized-partition device
   tree loaded correctly), `/proc/mtd` showed the full new layout, `/`/`/boot`/`/srv/data` all
   mounted correctly, all three RAID arrays healthy and unchanged, `systemd` started cleanly with
   no errors anywhere in the boot log. **`bootcmd` currently = `run bootcmd_nand` — the box now
   boots entirely from NAND**, independent of the SATA disks' partition tables.

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
`bootcmd`-style change (`run bootcmd_sata` or `run bootcmd_nand` at the prompt) before committing
to it. `fw_setenv` writes directly to NAND immediately — there's no "try before you save" step
like U-Boot's own `setenv` (RAM-only) → `saveenv` (persist) split. Every actual `bootcmd` cutover
done on this device so far (`bootcmd_sata`'s disk-mirror test, and the `bootcmd_nand` switch in
"Booting from NAND" above) has therefore been a real, `fw_setenv`-committed reboot test rather than
an unpersisted one — accepted given no serial access, with `bootcmd_sata` deliberately kept intact
as a fallback throughout. Use `fw_setenv` for variables you're already confident in (or just
reading current values); a real reboot test remains the only option here for anything touching the
boot chain itself.

## Base Setup

+ [forum.doozan.com](https://forum.doozan.com/read.php?2,12096)
+ [forum.doozan.com](https://forum.doozan.com/read.php?2,35331)
+ <https://minaret.biz/tips/ix2/index.html#introduction>

## Swap

+ [Linuxize: Add Swap Space on Debian 10](https://linuxize.com/post/how-to-add-swap-space-on-debian-10/)
+ [DigitalOcean: Add Swap Space on Debian 10](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-debian-10)
