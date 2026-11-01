# ix2-ng reference material (not this project's device)

The Iomega StorCenter ix2-ng and ix2-dl look nearly identical (same 2-bay enclosure, same Marvell
Kirkwood SoC/RAM, same EMC LifeLine OS) but differ fundamentally in boot architecture:

- **ix2-dl** (this project's device) has a 1GB onboard NAND flash chip. The OS lives on the NAND
  and boots independently of the hard drives — confirmed by this project's own captured logs
  (`artifacts/logs/`), which show UBI/UBIFS mounting the OS from NAND.
- **ix2-ng** has only a 1MB SPI flash chip and instead boots from boot sectors
  (`zImage`/`initrd`) written directly onto the primary hard drives. Recovering/reimaging it means
  byte-copying those images onto the drive's boot sectors from a secondary Linux machine.

The files in this directory (`zImage`, `initrd`, `boot.tgz`) are an EMC factory reimaging package
for the **ix2-ng disk-boot variant**. They are **not applicable** to this project's NAND-based
ix2-dl unit and should not be flashed to it. Kept here only for reference/comparison in case the
distinction becomes relevant later.

`boot.tgz` (247MB) is over GitHub's 100MB per-file limit, so it is git-ignored and kept only in
local checkouts; `zImage` and `initrd` are committed.
