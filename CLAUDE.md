# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a software project** — there is no source code, build system, linter, or test
suite. The goal of this project is to **install mainline Debian onto an Iomega StorCenter ix2-dl**
NAS, replacing the stock Iomega/EMC firmware. The repo is a personal archive of firmware images,
boot logs, a device configuration export, and notes supporting that goal.

The device is a Marvell Kirkwood platform: a Feroceon 88FR131 CPU (ARMv5TE, `DB-88F6282A-BP`),
with a **1GB onboard NAND flash chip**. The stock kernel was Linux 2.6.31.8, originally booting
independently of the hard drives — confirmed by this project's own captured logs
(`artifacts/logs/`), which show UBI/UBIFS mounting the stock OS from NAND. As of 2026-09-11 this
project actually boots that way again, but with its own kernel: `bootcmd` runs `bootcmd_nand`,
which reads a mainline-ish kernel (bodhi's community `linux-image-7.1.9-kirkwood-tld-1`, kirkwood-
tld build, with `kirkwood-lenovo-ix2-dl.dtb` appended) and a real Debian initramfs directly from
NAND via U-Boot's `bootm` — the original stock kernel/initrd/rootfs content in NAND was
deliberately overwritten to make room (see `docs/boot.md`'s "Booting from NAND" section for the
full story, including the device-tree edit that resized the NAND partitions to fit). Before that,
this same kernel booted via `bootcmd_sata` from the SATA disk's `/boot` partition instead — that
path is still intact and kept as the primary fallback (`bootcmd_sata` in `docs/boot.md`).

As of 2026-09-13, `/` and `/srv/data` also moved: originally degraded RAID1 arrays on the
original 1TB disk (`sdb`), they now live on separate GPT partitions (`sda1`, `sda2`) on the
second, 4TB disk (`sda`) — unmirrored for now, since only one 4TB drive is available so far (see
`docs/boot.md`'s "Root and /srv/data migration to sda" section). `/boot` is unaffected, still
`md0`. The old `sdb`-based arrays are kept intact, unwiped, as a rollback safety net.

**Important distinction:** the visually near-identical **ix2-ng** is a *different* device — same
enclosure and SoC, but only a 1MB SPI flash chip, booting instead from boot sectors written
directly onto the hard drives. It is not interchangeable with the ix2-dl. (Confusingly, the stock
ix2-dl firmware's own boot log self-reports `Model name is ix2-ng` internally — that's an EMC
firmware/codename quirk, not evidence of the actual hardware variant; the retail name, confirmed
via the `.icfg` export, is "Iomega StorCenter ix2-dl".) ix2-ng material is deliberately kept out of
`artifacts/` — see `reference/ix2-ng/`.

There are no commands to build, lint, or test — do not look for or invent a build system.

## Layout

- `README.md` — project overview and entry point.
- `docs/boot.md` — the primary reference. Contains the actual U-Boot shell commands used to:
  netboot/USB-boot a mainline kernel, build `uImage`/`uInitrd` from a raw `zImage` + device-tree
  blob via `mkimage`, and every boot chain variant this project has used or kept as a fallback
  (`bootcmd_nand` — current default, boots straight from NAND; `bootcmd_sata` — the SATA-disk
  path used before, kept intact as fallback; `bootcmd_usb`) including root filesystem label
  conventions (`LABEL=rootfs`, `LABEL=ROOT`) and `mtdparts`. Also documents the device-tree edit
  that resized NAND's `kernel`/`initrd` MTD partitions to make `bootcmd_nand` possible, and NAND
  archaeology findings from the original stock `boot` partition before it was overwritten. Also
  documents the later migration of `/` and `/srv/data` off the original disk's RAID1 arrays onto
  GPT partitions on the 4TB disk. Links to the doozan.com forum threads that are the upstream
  source for this process.
- `artifacts/logs/` — captured device state from the stock unit: `boot.txt`/`u-boot.txt` (U-Boot
  banner/boot log), `dmesg-no-disk.txt`/`dmesg-with-disk.txt` (stock 2.6.31.8 kernel boot log,
  captured with and without a SATA disk attached — not duplicates), `printenv.txt` (stock U-Boot
  environment variables), `nandconfig.txt` (NAND-specific env vars), `proc.txt` (`ps -ef` from the
  running stock firmware). Use these as the reference for "what the stock config looks like" when
  comparing against a device being reflashed.
- `artifacts/images/` — photos of the physical hardware (board/label shots), not diagrams.
- `artifacts/firmware/configuration.icfg` — an exported device configuration (iomega/EMC "icfg"
  format, XML) covering system identity, network, energy, email/alert, and other admin-panel
  settings. Useful as a reference for the config schema or for restoring settings post-reflash.
  Also in this directory: `uboot-*-dump.bin` (raw MTD partition backups, see `docs/boot.md`'s
  "NAND flash" section for what each one is and its actual recovery value), and
  `kirkwood-lenovo-ix2-dl.dtb`/`.dts` (original) plus `kirkwood-lenovo-ix2-dl-resized.dtb`/`.dts`
  (edited, currently deployed) — the device-tree source behind the current NAND partition layout
  and `bootcmd_nand` boot chain.
- `reference/ix2-ng/` — factory reimaging package (`zImage`, `initrd`, `boot.tgz`) for the
  **different, disk-boot ix2-ng device** — kept for comparison only, not applicable to this
  project's ix2-dl. See `reference/ix2-ng/README.md` for why it's split out.
- `ansible/` — a real, actively-used playbook (eleven roles) that provisions this NAS's Debian
  userspace end to end (packages, swap, security/firewall, tuning, RAID/SMART alerting via
  ntfy.sh, tailscale, NFS, Samba, package cleanup, disk-diagnostics binaries, media-library
  sorting tools) — see `ansible/README.md` for the full rundown of what each role does and why.
  Deliberately does **not** cover kernel/NAND/boot-chain changes (see `docs/boot.md` for those) —
  that's treated as manual, one-off, device-specific work, not idempotent config.

## Working with this repo

- Several files are large binaries (firmware images, `.tgz` archives, kernel/initrd blobs).
  Use `file <path>` before assuming something is text, and avoid opening large binaries with
  generic text tools.
- When editing or extending the boot notes in `docs/boot.md`, preserve the shell/U-Boot
  command-block style already used there (fenced ``` blocks of raw U-Boot console commands) rather
  than converting to prose.
- Do not merge `reference/ix2-ng/` content into `artifacts/` — they are different, incompatible
  hardware despite similar naming and identical enclosures.
