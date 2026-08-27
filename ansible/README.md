# Ansible playbook for the iomega NAS

Codifies the manual bring-up work from the initial install (packages, swap, hostname) plus a
follow-up review's security/reliability recommendations, so none of it lives only as one-off
commands typed over a serial console.

## Layout

Five roles, applied in this order (`playbook.yml`):

- **base** — utility packages (rsync, smartmontools, lm-sensors, gnupg), hostname.
- **swap** — swapfile creation and persistence.
- **security** — sudo/ufw/unattended-upgrades packages, admin user's sudo-group membership,
  explicit sshd policy, firewall rules.
- **tuning** — journald size cap, `vm.swappiness`, RAID resync floor, one-time `sensors-detect`
  (depends on `lm-sensors` from the `base` role, hence the ordering).
- **monitoring** — push notifications via [ntfy.sh](https://ntfy.sh) for mdadm RAID events (via
  its `PROGRAM` directive), smartd alerts (via `-M exec`, smartd configured against both disks by
  stable `/dev/disk/by-id` path with a weekly long self-test schedule), and an lm-sensors hardware
  alarm-bit check on a systemd timer. Depends on `smartmontools`/`lm-sensors` from `base`. **The
  real ntfy topic lives in `ansible/host_vars/iomega.yml` (gitignored, not this repo)** — it's a
  bearer secret: anyone who knows it can read your alerts or post fake ones on public ntfy.sh.
  `defaults/main.yml`'s placeholder deliberately fails loudly (curl 404s) if you forget to set it.

Each role's tunables live in its own `defaults/main.yml` (e.g. `roles/swap/defaults/main.yml` for
swapfile size/path) — override via `-e` or a host/group_vars file rather than editing the role.

## Usage

```bash
ansible-galaxy collection install ansible.posix   # provides the sysctl module used below
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```

You'll be prompted for **`admin`'s password** — `sudo` is installed and `admin` is in the `sudo`
group (done by hand on 2026-08-26, replacing the original bootstrap `stano` user, which no longer
exists on the box).

## What this does NOT cover (deliberately, needs a human decision first)

- **Kernel management.** Installing/updating bodhi's kirkwood-tld kernel and rebuilding
  `uImage`/`uInitrd` is a multi-step, device-specific process (see `../docs/boot.md`) involving a
  reboot and U-Boot interaction — not something to automate blind against production storage.
- **Adding the second disk to the RAID arrays** (`mdadm --add`). A live, carefully-sequenced task
  done by hand once the pending data copy off the old disk is confirmed complete — see the
  project's plan notes, not a routine idempotent playbook task.
- ~~RAID/SMART/sensor alerts~~ — **implemented, see the `monitoring` role above** (ntfy.sh push
  notifications, chosen 2026-08-26 over local-only mail or an external SMTP relay).

## OS release: trixie (upgraded from bookworm on 2026-08-27)

Upgraded in place over SSH (`apt upgrade --without-new-pkgs` then `apt full-upgrade`, staged per
Debian's documented bookworm->trixie procedure) — no NAND/U-Boot changes, no kernel change
(bodhi's third-party `7.1.9-kirkwood-tld-1` isn't a Debian-archive package, so apt never touches
it; the trixie release notes explicitly document third-party-kernel armel systems as a supported
upgrade path). `/etc/apt/sources.list` isn't managed by this playbook — updating it for a future
release still needs the same by-hand edit. Verified post-upgrade: same kernel, all three `mdX`
arrays reassembled in their prior degraded state, ssh/ufw/cron/smartd all healthy, `fw_printenv`
round-trips correctly.

**Important: trixie is upstream's last release to support the armel architecture at all.** This
board's CPU (Feroceon 88FR131, ARMv5TE) has no hardware FPU, so it can't move to armhf/arm64 —
Debian's suggested migration target for other armel hardware doesn't apply here. There is no
further OS upgrade path for this device after trixie; plan hardware replacement/retirement with
that ceiling in mind.

The previous "no armel-binary security-patch channel" gap (bookworm-security had dropped its
`binary-armel` index once bookworm aged into `oldstable`) no longer applies: trixie-security
serves a normal, complete `armel` Packages index (confirmed live against
`https://security.debian.org/debian-security/dists/trixie-security/Release` — lists armel
alongside `amd64 arm64 armhf i386 ppc64el riscv64 s390x`), so the `security` role's old
`[arch=armhf]` sources.list workaround was removed as dead weight.
