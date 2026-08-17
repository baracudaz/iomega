# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not a software project** — there is no source code, build system, linter, or test
suite. The goal of this project is to **install mainline Debian onto an Iomega StorCenter ix2-dl**
NAS, replacing the stock Iomega/EMC firmware. The repo is a personal archive of firmware images,
boot logs, a device configuration export, and notes supporting that goal.

The device is a Marvell Kirkwood platform: a Feroceon 88FR131 CPU (ARMv5TE, `DB-88F6282A-BP`),
with a **1GB onboard NAND flash chip** that holds the OS and boots independently of the hard
drives — confirmed by this project's own captured logs (`artifacts/logs/`), which show UBI/UBIFS
mounting the stock OS from NAND. The stock kernel is Linux 2.6.31.8; the goal of the
notes/artifacts here is booting a modern mainline kernel (5.x, `kirkwood-lenovo-ix2-dl.dtb`) with
an initramfs via U-Boot's `bootm`, either from USB/ext2 or NAND.

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
  blob via `mkimage`, and the working NAND boot chain (`bootcmd_usb` / `bootcmd_sata`) including
  root filesystem label conventions (`LABEL=rootfs`, `LABEL=ROOT`) and `mtdparts`. Links to the
  doozan.com forum threads that are the upstream source for this process.
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
- `reference/ix2-ng/` — factory reimaging package (`zImage`, `initrd`, `boot.tgz`) for the
  **different, disk-boot ix2-ng device** — kept for comparison only, not applicable to this
  project's ix2-dl. See `reference/ix2-ng/README.md` for why it's split out.
- `ansible/` — placeholder for future Debian provisioning playbooks; empty for now.

## Working with this repo

- Several files are large binaries (firmware images, `.tgz` archives, kernel/initrd blobs).
  Use `file <path>` before assuming something is text, and avoid opening large binaries with
  generic text tools.
- When editing or extending the boot notes in `docs/boot.md`, preserve the shell/U-Boot
  command-block style already used there (fenced ``` blocks of raw U-Boot console commands) rather
  than converting to prose.
- Do not merge `reference/ix2-ng/` content into `artifacts/` — they are different, incompatible
  hardware despite similar naming and identical enclosures.
- The git repository currently has no commits — check `git status`/`git log` before assuming any
  history exists.
