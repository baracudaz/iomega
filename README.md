# iomega

Installing mainline Debian on an **Iomega StorCenter ix2-dl** NAS, in place of the stock
Iomega/EMC firmware.

The device is a Marvell Kirkwood board (Feroceon 88FR131, ARMv5TE) that ships with Linux 2.6.31.8
and boots from a 1GB onboard NAND flash chip. Its stock firmware self-identifies internally as
`ix2-ng` in some log strings, but that's an EMC firmware/codename quirk — the retail device, and
its actual NAND-based boot architecture, is the ix2-dl. The visually identical **ix2-ng** is a
genuinely different, disk-boot device and is not interchangeable with this one; see
[`reference/ix2-ng/`](reference/ix2-ng/) for details.

## Layout

- [`docs/`](docs/) — boot/install instructions (U-Boot commands, image build steps).
- [`artifacts/`](artifacts/) — captured logs, hardware photos, and firmware images pulled from this
  project's actual ix2-dl device, kept for reference while reflashing.
- [`reference/ix2-ng/`](reference/ix2-ng/) — factory reimaging material for the different,
  disk-boot ix2-ng device — comparison reference only, not applicable to this ix2-dl.
- [`ansible/`](ansible/) — the playbook (eleven roles) that provisions the Debian userspace: packages,
  firewall, RAID/SMART alerting, tailscale, NFS, Samba, and more. See
  [`ansible/README.md`](ansible/README.md).

## Getting started

See [`docs/boot.md`](docs/boot.md) for the U-Boot boot chain and image build steps.

## License

The notes, docs, and Ansible playbook are [MIT](LICENSE)-licensed. That doesn't extend to
third-party material kept here for reference: the EMC/Iomega firmware images and dumps under
`artifacts/firmware/` and `reference/ix2-ng/` belong to their respective owners, and the
device-tree files are derived from the Linux kernel (GPL-2.0).
