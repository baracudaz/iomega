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

## Known gap: no armel-binary security-patch channel

`bookworm-security` dropped `armel`'s `binary-armel` index once bookworm aged into `oldstable`
(confirmed 2026-08-26 against the live Release file — it now lists only
`amd64 arm64 armhf i386 ppc64el`). By default this means apt fetches *nothing at all* from that
repo (`Architecture: all` packages, like `ca-certificates`, are bundled inside each per-arch
`Packages` file — there's no separate universal index to fall back to). The `security` role's
sources.list task restricts that source to exactly one small foreign architecture (`armhf`,
~300KB) purely to pull that file and its `all`-arch security patches cheaply — **do not add more
architectures to that list**: apt's `arch=` option fetches every listed architecture
unconditionally (not just whichever intersect with what's needed), so listing all 5 real
architectures wastes ~1.6MB per `apt update` for zero extra benefit (confirmed the hard way).
What this doesn't fix: architecture-specific *compiled* packages (kernel, libc, openssl, etc.)
still have no official security-patch channel on this release/architecture combination at all —
`unattended-upgrades` also pulls from `bookworm-updates` as a partial mitigation for those (urgent
non-security fixes, not CVE patches). This device's security posture leans more heavily on `ufw`
(deny-all except SSH) and not exposing services than on OS patching for that remaining gap.
