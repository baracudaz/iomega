# Ansible playbook for the iomega NAS

Codifies the manual bring-up work from the initial install (packages, swap, hostname) plus a
follow-up review's security/reliability recommendations, so none of it lives only as one-off
commands typed over a serial console.

## Layout

Ten roles, applied in this order (`playbook.yml`):

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
- **motd** — installs [fastfetch](https://github.com/fastfetch-cli/fastfetch) as the SSH login
  banner, dropped in as `/etc/update-motd.d/10-fastfetch`. Chosen over neofetch (archived upstream
  as of 2024, and a bash script that forks a separate external command per data point) — fastfetch
  is a single actively-maintained C binary, which matters on this box's single 400-BogoMIPS core.
  Relies on Debian's stock `/etc/pam.d/sshd` (ships `session optional pam_motd.so
  motd=/run/motd.dynamic`, which runs every `/etc/update-motd.d/*` script per login and caches the
  result) rather than touching `pam.d` or `sshd_config` itself — nothing added here risks the box's
  only remote access path. `roles/motd/templates/config.jsonc.j2` picks NAS-relevant modules (OS,
  kernel, uptime, packages, CPU/memory/swap, `/` and `/srv/data` usage, LAN IP) over fastfetch's
  fancier but irrelevant defaults (GPU, battery, media player). Verified live over a real interactive SSH login (not just
  running the script by hand) — the banner renders before the shell prompt, as expected.
- **tailscale** — installs the self-built `tailscale`/`tailscaled` binaries (pinned version +
  checksum in `defaults/main.yml`) from a GitHub Release, deploys the systemd unit and
  `/etc/default/tailscaled`, enables the service. Deliberately does **not** run `tailscale up` —
  see below.
- **nfs** — installs `nfs-kernel-server`, exports `nfs_shared_dir` (`/srv/data/shared`) directly
  as the NFSv4 pseudo-root (`fsid=0`, so clients mount `iomega:/` and see its contents right at
  the mount root), restricted to `nfs_allowed_subnet` (derived from the box's actual
  default-route interface at run time, not hardcoded — see `roles/nfs/defaults/main.yml`) in
  `/etc/exports`. The export deliberately covers only that one directory, not the wider
  `/srv/data` — `/srv/data/<user>` (private per-user Samba home dirs, see the `samba` role
  below) live as siblings of it and must stay outside any NFS export: NFS exports are
  subtree-based (no "export this dir but not that subdirectory" option), so exporting a wider
  parent would make those private dirs NFS-visible too. NFSv3 is explicitly
  disabled (`/etc/nfs.conf.d/nfsv4-only.conf`, confirmed live via
  `/proc/fs/nfsd/versions` → `-3 +4 +4.1 +4.2`) so only port 2049 is actually needed for NFS
  traffic. Disabling v3 in `nfs.conf` alone does **not** stop `rpcbind`/`rpc.statd` from
  running though — Debian's `nfs-server.service` pulls them in via `Wants=` regardless of
  the configured versions — so both are masked outright (`systemctl mask`, survives
  `nfs-server` restarts, unlike a plain stop). Neither is needed for NFSv4-only serving: no
  NLM crash-recovery (`rpc-statd` is v3-only), no RPC portmapper lookup (`rpcbind` — NFSv4
  uses a fixed well-known port). **This took two rounds to get right**: an early real macOS
  mount failed with `rpcbind` masked and worked once unmasked, which looked like solid proof
  macOS's client needs it unlike Linux's — but that test predated finding the actual fix for
  macOS NFS mounting (see "macOS NFS client" below). Once that fix was in place, re-tested
  with `rpcbind` masked again: a fresh macOS mount still negotiated genuine `vers=4.0` with
  zero traffic reaching port 111 at all. It was never actually required — the earlier test was
  confounded by not having the real client-side fix yet. `rpc.mountd` is left alone either
  way — it's a hard `Requires=` of `nfs-server.service` regardless of the v3 setting; masking
  it breaks the server outright. `ufw` allows port 2049 from two sources:
  the LAN subnet, and separately anything arriving over the `tailscale0` interface (not a
  Tailscale CGNAT IP-range match — that range is shared across unrelated deployments, so
  the interface itself is the more precise restriction). SSH already worked over the
  tailnet without any extra rule (`ufw allow OpenSSH` has no source restriction, and
  Tailscale doesn't bypass the host firewall — tailnet traffic still passes through the
  normal ufw chain like any other interface).

  **Write access, verified properly** (not just mount+read, which earlier passes in this
  same session had wrongly implied was a full test): `/srv/data` and its existing
  `timecapsule-backup` dir are `root:root` `755` — writable by nobody over NFS by design,
  since `root_squash` (the default, deliberately not disabled — see `nfs_exports` above) maps
  a client's root to `nobody`. Tested against a scratch `0777` directory instead, from macOS,
  as a normal (non-`sudo`) user: write, readback, and correct ownership (the real client
  uid/gid, unsquashed — only root gets remapped) all confirmed working end-to-end. `sudo
  touch` at the mount root still correctly fails (`root_squash` doing its job, not a bug).
  Takeaway for actually using this export to write data: create a subdirectory owned by
  (or permissioned for) whichever user will do the writing — don't reach for
  `no_root_squash` to work around it. The role now creates exactly that: `nfs_shared_dir`
  (`/srv/data/shared`), `/tmp`-style sticky-bit world-writable (`1777`) rather than owned by
  one specific UID — this is a multi-person household NAS (the old Time Capsule disk's
  `ShareRoot/Users/` had folders for several family members, not one account), so matching a
  single client UID would've been the wrong shape. Sticky bit means anyone can create a file,
  but only its own owner (or root) can delete/rename it — one person's copy job can't clobber
  another's.
- **samba** — installs `samba` for macOS Finder and CIFS-mounting Linux clients (e.g. the rpi
  running k3s), with a two-tier IAM model built on top of a single unified data tree (one
  directory can be reached over either protocol, not separate copies per protocol — matching
  how NAS vendors like Synology/QNAP actually structure this):
  - **`[shared]`** (`samba_share_path`, `/srv/data/shared`) — the same directory the `nfs` role
    exports. Anonymous, read-write: `guest ok = yes` in `smb.conf`, backed globally by
    `map to guest = Bad User` / `guest account = nobody` (the former is what actually lets an
    anonymous/no-credentials session be accepted at all, rather than rejected before Samba
    reaches the per-share `guest ok` check). Still the same `1777` sticky-bit world-writable
    directory as before.
  - **`[homes]`** — Samba's special per-user share (`%S` expands to the connecting username, so
    every account in `samba_users` automatically gets exactly its own share with no per-user
    config block needed). Each user's Unix home is `samba_users_home_root/<name>`
    (`/srv/data/<name>`), created `0700`, owned by that user, `valid users = %S` so no other
    user (or guest) can reach it. **Deliberately Samba-only, never in `/etc/exports`**: plain
    NFS (`sec=sys`, what the `nfs` role uses) trusts a client's claimed UID with no real
    authentication, so a "private" NFS home dir would only be private by convention, not
    enforcement — Samba's privacy here is real, because it actually checks a password.
  - **`[TimeMachine]`** — a Time Machine backup destination for macOS clients. One fixed share
    name (`samba_timemachine_share_name`) rather than a `%S`-style special share like `[homes]`
    — Avahi's static Bonjour advertisement (below) needs one stable name to point at — but the
    path is still `%u`-substituted (`samba_timemachine_root/%u`, i.e. `/srv/data/timemachine/<name>`,
    `0700`, owned by that user), so each user's backup sparsebundle stays private and separate
    from their general files in `[homes]`. `fruit:time machine = yes` is the flag macOS actually
    checks to offer it as a backup destination; `[global]`'s `vfs objects = catia fruit
    streams_xattr` + `fruit:aapl = yes` (applied to every share) is what provides that plus
    general Apple metadata/resource-fork support — all three VFS modules already ship in
    Debian's `samba` package, no extra package needed. Uncapped by default
    (`samba_timemachine_max_size`, e.g. `"1T"`, to bound it).
  - **Bonjour/mDNS auto-discovery** — `avahi-daemon` is installed and given a service
    definition (`templates/avahi-smb.service.j2`) advertising `_smb._tcp`, `_device-info._tcp`,
    and `_adisk._tcp` (the `dk0=adVN=<TimeMachine share>,adVF=0x82` TXT record is what makes
    System Settings → Time Machine → Add Backup Disk list this box automatically, without a
    manual connect first). `ufw` allows mDNS (UDP 5353) from the LAN subnet only — mDNS is
    link-local multicast and doesn't route over the tailnet, so there's no `tailscale0`
    equivalent; a remote Mac can still connect to `[TimeMachine]` manually by hostname/IP over
    Tailscale, just without auto-discovery.
  - Accounts are defined in `samba_users` (`roles/samba/defaults/main.yml`), each getting both a
    Unix system account (nologin, home dir only ever used as its Samba share target, never for
    login) and a Samba password set via `smbpasswd`. **Real passwords live in
    `ansible/host_vars/iomega.yml`** (gitignored, same pattern as `monitoring_ntfy_topic`) — the
    shipped placeholder deliberately fails an `assert` rather than silently provisioning a share
    with a known password.
  - SMB is restricted to SMB2/3 only (no legacy SMB1). `nmbd` (NetBIOS name service) is masked
    outright — unneeded since macOS uses mDNS/Bonjour (now genuinely served by `avahi-daemon`,
    not `nmbd`) and a Linux cifs mount is given an IP/hostname directly, neither relies on
    NetBIOS browsing. Port 445 is firewalled via the same LAN-subnet-plus-`tailscale0`-interface
    `ufw` pattern as the `nfs` role; see the Bonjour bullet above for the separate mDNS rule.
  - Note: `smbpasswd` has no idempotent "change if different" mode, so rotating a password in
    `host_vars` after the account already exists needs a manual `smbpasswd` run on the box — the
    playbook only sets it the first time an account is created. For the k3s side, point a CSI
    driver (e.g. `csi-driver-smb`) or a plain `mount -t cifs` at `//iomega/shared` (anonymous) or
    `//iomega/<user>` (credentials in a Secret).
- **cleanup** — purges a defined list of unnecessary/dead-weight packages (reportbug, hardware
  auto-detection tooling, an unused local mail daemon that appeared as an upgrade side effect,
  etc.), then `autoremove`s whatever that orphans. Every entry's reasoning — and what was
  deliberately left out and why — lives in `roles/cleanup/defaults/main.yml`, not here.

## macOS NFS client: Finder's "Connect to Server" — fixed

**Working recipe (one-time setup per Mac, then plain Finder "Connect to Server" works normally
from then on):**

1. `/etc/nfs.conf` must contain `nfs.client.mount.options = vers=4` (`sudo` to edit).
2. Run `sudo automount -vc` once after that edit — **this step is what earlier attempts missed.**
   Just editing `nfs.conf` isn't enough; something in the NetFS/automount stack needs the explicit
   refresh to pick it up (macOS appears to cache per-host connection state — see the
   `ServerMarkers`/`GetKnownServerMarker` log lines from debugging this — and a plain file edit
   doesn't invalidate that cache).
3. Finder → Go → Connect to Server → `nfs://192.168.0.199/` → Connect. Plain URL, no path
   suffix, no `vers=4,` prefix trick, no colon-before-path variant — none of that was ever
   necessary. **No `/shared` suffix**: the NFS export root *is* `nfs_shared_dir`
   (`/srv/data/shared`) directly (see the `nfs` role bullet above) — appending `/shared` looks
   for a subdirectory that doesn't exist under the export and fails with "No such file or
   directory".

Verified live: `nfsstat -m` after this shows genuine `vers=4.0,tcp,port=2049`; mounts at
`/Volumes/<mount-name>`; real read/write confirmed. Reproduced twice (fresh unmount, fresh
`automount -vc`, fresh Finder connect) to confirm it's not a one-off.

**How this was found**, kept for context since the debugging path itself has real lessons: this
was extensively misdiagnosed as unfixable before landing on the actual answer. One real server-side
bug got fixed along the way even though it turned out not to be the actual cause: `ufw` was
silently dropping all traffic to `rpcbind` (111), worth having fixed regardless, but `rpcbind`
itself later turned out to be unnecessary entirely (masked outright now — see the `nfs` role
bullet above). The export's `secure` flag (switched to `insecure`) genuinely is necessary, since
Finder can't bind a reserved port either way — but not sufficient alone. Also ruled out:
mounting a subpath vs. the export root (irrelevant), the
`nfs://vers=4,host:/path` URL option-embedding trick (a 2017-era OS X 10.11 quirk, not applicable
here), a colon-before-path URL variant (not it either). The macOS automounter
(`/etc/auto_master`/`/etc/auto_nfs`, `nolocallocks` being the key detail there — without it a
`vers=4` mount silently falls back to v3, which fails since this server has v3 disabled) was also
confirmed working as an alternative, then deliberately reverted at the user's preference before
the real Finder fix was found. **The one genuinely wrong conclusion drawn mid-debugging**: that
`/etc/nfs.conf`'s `nfs.client.mount.options` was "confirmed irrelevant" to Finder's client. It
isn't irrelevant — it was just never tested together with `sudo automount -vc` in the same pass.

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

Each role in `playbook.yml` is tagged with its own name — use `--tags <role>` (e.g.
`--tags nfs`) to run just one while iterating. Worth knowing given how slow a full run can be
on this hardware: the `tailscale` role's `get_url` checksum verification runs *on the box
itself* (234MB RAM, ~400 BogoMIPS single core), not the control machine — downloading and
hashing ~47MB there routinely takes several minutes even when nothing's actually changed.

## What this does NOT cover (deliberately, needs a human decision first)

- **Kernel management.** Installing/updating bodhi's kirkwood-tld kernel and rebuilding
  `uImage`/`uInitrd` is a multi-step, device-specific process (see `../docs/boot.md`) involving a
  reboot and U-Boot interaction — not something to automate blind against production storage.
- **Adding the second disk to the RAID arrays** (`mdadm --add`). A live, carefully-sequenced task
  done by hand once the pending data copy off the old disk is confirmed complete — see the
  project's plan notes, not a routine idempotent playbook task.
- ~~RAID/SMART/sensor alerts~~ — **implemented, see the `monitoring` role above** (ntfy.sh push
  notifications, chosen 2026-08-26 over local-only mail or an external SMTP relay).
- **Tailscale login.** The `tailscale` role installs and runs `tailscaled`, but deliberately does
  **not** run `tailscale up` — that needs a human to open a one-time login URL (or a
  pre-generated `--authkey`, which this role doesn't manage). Run it by hand after the playbook:
  `sudo tailscale up`.
- **Bumping the Tailscale version.** `tailscale_version` + its two checksums in
  `roles/tailscale/defaults/main.yml` are pinned by hand, not auto-tracked from the upstream
  build. The fork that builds them
  ([`baracudaz/tailscale-builds`](https://github.com/baracudaz/tailscale-builds)) auto-builds new
  upstream stable releases daily, but pulling a new one onto this box is still a deliberate,
  manual version bump — a root-level network daemon isn't something to auto-upgrade blind.

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
