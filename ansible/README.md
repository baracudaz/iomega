# Ansible playbook for the iomega NAS

Codifies the manual bring-up work from the initial install (packages, swap, hostname) plus a
follow-up review's security/reliability recommendations, so none of it lives only as one-off
commands typed over a serial console.

## Layout

Four roles, applied in this order (`playbook.yml`):

- **base** — utility packages (rsync, smartmontools, lm-sensors), hostname.
- **swap** — swapfile creation and persistence.
- **security** — sudo/ufw/unattended-upgrades packages, admin user's sudo-group membership,
  explicit sshd policy, firewall rules.
- **tuning** — journald size cap, `vm.swappiness`, RAID resync floor, one-time `sensors-detect`
  (depends on `lm-sensors` from the `base` role, hence the ordering).

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
- **RAID/SMART email alerts.** `mdadm.conf` already has `MAILADDR root` and `smartd` is installed,
  but there's no mail transport agent on this box, so alerts currently go nowhere. Wiring up real
  delivery needs a decision about *how* (an external SMTP relay's credentials for something like
  `msmtp`, vs. local-only delivery you'd have to remember to check) — not something to invent
  without asking.
