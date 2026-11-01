# Ansible playbook for the iomega NAS

Codifies the manual bring-up work from the initial install (packages, swap, hostname) plus a
follow-up review's security/reliability recommendations, so none of it lives only as one-off
commands typed over a serial console.

## Layout

Eleven roles defined, in this order (`playbook.yml`), all applied:

- **base** — generic bootstrap: utility packages (rsync, smartmontools, lm-sensors, gnupg,
  sysstat, testdisk, tmux), `--force-unsafe-io` for dpkg (speed on this weak CPU), hostname.
  testdisk/tmux were added 2026-09-10 after a manual `apt-get install testdisk` on the box to
  investigate a repurposed 4TB drive (`ST4000VN000-1H41`, added as `/dev/sda`) that showed as
  effectively empty (4.4MB used of 3.7TB, only Windows `System Volume Information` visible) —
  raw sector sampling found dense non-zero data up to ~2TB in, consistent with a quick format
  rather than a wipe, so photorec (same Debian package as testdisk) is expected to be used for
  an actual carve. Tracked here so these tools survive a reprovision instead of being manual,
  undocumented installs; tmux is what lets photorec's ncurses TUI be driven/monitored
  non-interactively over SSH (`tmux capture-pane -e` preserves the ANSI highlight codes needed
  to read menu selection state).
- **uboot** — split out from `base` 2026-09-13 for cohesion (NAND/U-Boot-adjacent tooling vs.
  generic bootstrap). Installs `u-boot-tools` (`mkimage`); configures `fw_env.config` against
  the correct MTD env partitions; a compat symlink working around an upstream `initramfs-tools`
  bug that breaks `update-initramfs` outright on armel (its `kmod` hook hardcodes a
  `*-linux-gnu` glob that never matches armel's `-linux-gnueabi` triplet); forces
  `COMPRESS=gzip` in `initramfs.conf`; and installs a `/etc/initramfs/post-update.d/` hook that
  auto-builds (but deliberately does not auto-deploy to NAND) a fresh `uInitrd` after every
  `update-initramfs` run — see `docs/boot.md`'s NAND section for why the actual NAND write stays
  a manual, reviewed step on this device.
- **swap** — swapfile creation and persistence.
- **security** — sudo/ufw packages, admin user's sudo-group membership, explicit sshd policy,
  firewall rules. `unattended-upgrades` was installed/enabled here until 2026-09-13, when its
  background apt runs were found visibly hogging CPU on this box's single weak ARMv5 core during
  other work (see `docs/boot.md`) — removed and purged manually; manual `apt upgrade` is used
  instead. No ansible task re-removes it (nothing else in this playbook would reinstall it); if
  it's ever wanted back, its old config is recoverable from git history (`git log -p` on this
  role).
- **tuning** — journald size cap, `vm.swappiness`, RAID resync floor, one-time `sensors-detect`
  (depends on `lm-sensors` from the `base` role, hence the ordering).
- **monitoring** — push notifications via [ntfy.sh](https://ntfy.sh) for mdadm RAID events (via
  its `PROGRAM` directive), smartd alerts (via `-M exec`, smartd configured against both disks by
  stable `/dev/disk/by-id` path with a weekly long self-test schedule), and an lm-sensors hardware
  alarm-bit check on a systemd timer. Depends on `smartmontools`/`lm-sensors` from `base`. **The
  real ntfy topic lives in `ansible/host_vars/iomega.yml` (gitignored, not this repo)** — it's a
  bearer secret: anyone who knows it can read your alerts or post fake ones on public ntfy.sh.
  `defaults/main.yml`'s placeholder is rejected by an `assert` at the top of the role — ntfy.sh
  itself accepts any topic name, so without that check a forgotten topic would silently publish
  alerts to a public `CHANGE_ME_SEE_HOST_VARS` topic.

**No `motd` role (dropped 2026-09-11).** A fastfetch-based dynamic SSH login banner existed
briefly but was removed after it turned out to be *why* SSH logins felt slow: Debian's stock
`pam_motd` re-runs every `/etc/update-motd.d/*` script on every login, synchronously, before the
shell prompt appears — and the banner's `PublicIP`/`Weather` modules were live third-party API
calls sitting right in that critical path (confirmed live: ~1.1-1.6s for a real login vs. ~0.9s
for the bare SSH handshake alone; login is back to ~0.85-0.97s with the role gone). A
caching/background-refresh redesign was drafted but the simpler fix — no banner at all — is what
actually got deployed. fastfetch and its `libyyjson0` dependency were purged from the box.
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
- **samba** — **re-enabled 2026-09-11**, reversing a 2026-09-10 decision to disable it for
  being too heavy for this box's limited hardware (single-core ARMv5, 234MB RAM), especially
  once Avahi/mDNS and the AD-DC-capable `samba` package pull in their full dependency trees.
  **Slimmed down 2026-09-14** in exactly that direction: `avahi-daemon` (+`libnss-mdns`),
  `winbind` (+`libnss-winbind`/`libpam-winbind`) and `samba-ad-dc` were all purged, and the
  `samba` install now uses `install_recommends: false` so they don't creep back — `winbind` and
  `samba-ad-dc` are *Recommends* of `samba`, not Depends, and are pure waste on a standalone
  `security = USER` server with no domain to join. Between them `winbindd`, its `wb[IOMEGA]`
  worker, `samba-dcerpcd` and two idle `rpcd_lsad` processes were holding ~50-70MB RSS on a
  234MB box; after the purge `free -m` went from 82MB used / 151MB available to 71MB / 162MB,
  and all of those processes are gone. Two accepted trade-offs, both reversible: the Time
  Machine destination no longer auto-appears under System Settings → Time Machine → Add Backup
  Disk (mount the share in Finder first, then pick it there), and `iomega.local` no longer
  resolves — use the LAN IP or the Tailscale name. Removing them needed care with
  `/etc/nsswitch.conf`, which referenced both `winbind` (passwd/group) and `mdns4_minimal`
  (hosts); the package postrm scripts cleaned both correctly, verified afterwards.
  If Samba itself ever turns out too heavy again: `smbd` can be stopped and disabled and the
  `samba` package purged, along with the `ufw` rule for 445, without touching
  `/srv/data/shared`, `/srv/data/<user>`, or `/srv/data/timemachine` — none of that data lives
  in the serving software.

  **Throughput, measured 2026-09-15** (2GB cold-cache reads of the same file, so the numbers
  are comparable): local read off `md2` **93.6 MB/s**; raw TCP NAS→client (`/dev/zero`, no disk,
  no Samba) **34.3 MB/s**; SMB **23.2 MB/s** before `use sendfile`, **24.5 MB/s** after. The
  conclusion is the raw TCP figure: *this CPU can't drive its own gigabit NIC past ~34MB/s*
  (~275Mbps), with all offloads already on and the governor already at `performance`/1.6GHz.
  So the network stack — not Samba, and emphatically not the disks — is what caps this box, and
  Samba at 71% of the achievable ceiling has little left to give. `vmstat` during a transfer
  shows 82-98% *system* CPU with 0-6% idle, which is the same story from the other side.
  Ruled out by checking, each already optimal: CPU governor (`performance` is the only one
  `kirkwood-cpufreq` offers), readahead (8MB on all four devices), NIC offloads (rx/tx
  checksum, scatter-gather, TSO, GSO, GRO all on). The one lever with real upside left is
  **jumbo frames** — the driver reports `maxmtu 9500`, and fewer/larger packets directly
  attacks the per-packet CPU cost that *is* the bottleneck — but that needs the switch and
  every client to agree, so it's a network-wide change rather than something to set here. Installs `samba` for macOS Finder and CIFS-mounting Linux clients (e.g. the rpi
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
  - **Bonjour/mDNS auto-discovery — removed 2026-09-14.** `avahi-daemon` used to be installed
    with a service definition advertising `_smb._tcp`, `_device-info._tcp` and `_adisk._tcp` (the
    `dk0=adVN=<TimeMachine share>,adVF=0x82` TXT record is what made System Settings → Time
    Machine → Add Backup Disk list this box automatically, without a manual connect first). It
    was dropped as part of the memory slim-down above; the template and its `ufw` 5353 rule went
    with it. Connect via LAN IP or Tailscale name, and add the TM destination by mounting the
    share in Finder first. Git history has the old tasks and template if it's ever wanted back.
  - Accounts are defined in `samba_users` (`roles/samba/defaults/main.yml`), each getting both a
    Unix system account (nologin, home dir only ever used as its Samba share target, never for
    login) and a Samba password set via `smbpasswd`. **Real passwords live in
    `ansible/host_vars/iomega.yml`** (gitignored, same pattern as `monitoring_ntfy_topic`) — the
    shipped placeholder deliberately fails an `assert` rather than silently provisioning a share
    with a known password.
  - SMB is restricted to SMB2/3 only (no legacy SMB1). `nmbd` (NetBIOS name service) is masked
    outright — a Linux cifs mount is given an IP/hostname directly and macOS doesn't rely on
    NetBIOS browsing either, so nothing here needs it. Port 445 is firewalled via the same
    LAN-subnet-plus-`tailscale0`-interface `ufw` pattern as the `nfs` role.
  - Note: `smbpasswd` has no idempotent "change if different" mode, so rotating a password in
    `host_vars` after the account already exists needs a manual `smbpasswd` run on the box — the
    playbook only sets it the first time an account is created. For the k3s side, point a CSI
    driver (e.g. `csi-driver-smb`) or a plain `mount -t cifs` at `//iomega/shared` (anonymous) or
    `//iomega/<user>` (credentials in a Secret).
- **cleanup** — purges a defined list of unnecessary/dead-weight packages (reportbug, hardware
  auto-detection tooling, an unused local mail daemon that appeared as an upgrade side effect,
  etc.), then `autoremove`s whatever that orphans. Every entry's reasoning — and what was
  deliberately left out and why — lives in `roles/cleanup/defaults/main.yml`, not here.
- **watch** — deploys three sibling single-host diagnostics TUIs by the same author:
  [matthart1983/diskwatch](https://github.com/matthart1983/diskwatch) (disk — unifies
  `iostat`/`smartctl`/`lsblk`/`df` into one dashboard),
  [matthart1983/syswatch](https://github.com/matthart1983/syswatch) (system —
  CPU/memory/disks/processes/power/services), and
  [matthart1983/netwatch](https://github.com/matthart1983/netwatch) (network — `pcap`-based
  traffic inspection, connection attribution), to `/usr/local/bin/{diskwatch,syswatch,netwatch}`.
  One role rather than three: same deploy pattern, same reason for existing, same story — worth
  keeping together instead of duplicating the same explanation. Fetched from each project's
  GitHub releases (`get_url` + a pinned `sha256` on the tarball, `unarchive` to extract, same
  general shape as the `tailscale` role) — as of 2026-09-14 all three publish an
  `armv5te` Linux asset directly, so no self-building is needed any more. That wasn't always
  true: diskwatch/syswatch initially had **no upstream release for this hardware** (both only
  shipped `aarch64`/`x86_64`) and were hand-built via `cross build --target
  armv5te-unknown-linux-gnueabi` and checked into the repo as binaries; netwatch was further
  blocked by a **hard architecture wall**, not just a missing build — its source used
  `std::sync::atomic::AtomicU64` directly, and the `armv5te-unknown-linux-gnueabi` Rust target
  has no native 64-bit atomic support (`target_has_atomic` tops out at 32-bit on this CPU
  generation). Both limitations were resolved upstream since (see git history for the
  hand-built-binary era, if curious). Verified live: `diskwatch --diag` (its non-interactive
  dump mode) correctly reads both real disks (`sda`/`sdb`) and all `mdX` arrays; `syswatch`, run
  in a scratch `tmux` session, correctly rendered real CPU/memory/process data. Bump
  `*_version`/`*_tarball_sha256` in `roles/watch/defaults/main.yml` for a future version — see
  that file's comment for how to get the new checksum.
- **mediasort — removed 2026-09-13.** Installed `python3-mutagen`, `mediainfo`,
  `libimage-exiftool-perl`, and `jdupes` to support sorting the one-off PhotoRec carve output
  from `/srv/data/shared/recovery` into a `Music`/`Photos`/`Documents` tree — that sort completed
  (86,323 files organized, 3,597 duplicates removed) and the role/packages were no longer needed,
  so both were removed rather than left as dead weight.
  and confirmed running on the live host.

## macOS NFS client: Finder's "Connect to Server" — fixed

**Working recipe (one-time setup per Mac, then plain Finder "Connect to Server" works normally
from then on):**

1. `/etc/nfs.conf` must contain `nfs.client.mount.options = vers=4` (`sudo` to edit).
2. Run `sudo automount -vc` once after that edit — **this step is what earlier attempts missed.**
   Just editing `nfs.conf` isn't enough; something in the NetFS/automount stack needs the explicit
   refresh to pick it up (macOS appears to cache per-host connection state — see the
   `ServerMarkers`/`GetKnownServerMarker` log lines from debugging this — and a plain file edit
   doesn't invalidate that cache).
3. Finder → Go → Connect to Server → `nfs://<NAS address>/` (its current LAN IP or Tailscale
   name — no mDNS, see `docs/boot.md`'s "Network" section) → Connect. Plain URL, no path
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

## The CESA crypto engine: investigated 2026-09-15, deliberately unused

Boot logs show `marvell-cesa f1030000.crypto: CESA device successfully registered`, which
naturally invites "can we offload crypto to it?" **Answer: no, and it isn't close.** Written down
so nobody re-runs this investigation.

**It registers 10 algorithms, and none of them are what this box actually uses:**

| | CESA offers | what this box negotiates |
| --- | --- | --- |
| ciphers | `cbc/ecb(aes)`, `cbc/ecb(des)`, `cbc/ecb(des3_ede)` | ChaCha20-Poly1305 (SSH default, WireGuard/Tailscale), AES-**GCM**/CCM (SMB3), AES-**CTR** (SSH fallback) |
| hashes | `sha1`, `md5`, `hmac(sha1)`, `hmac(md5)` — all at **priority 0**, so software wins anyway | HMAC-SHA256, AES-GMAC/CMAC |

CESA is a 2010-era engine and everything here has moved to AEAD modes it predates. Forcing a
match would mean downgrading to deprecated CBC ciphers — worse security, for the numbers below.

**Even in its best case (AES-128-CBC, which it does support) it mostly loses.** `openssl speed`,
software vs. the `afalg` engine, 1000s of bytes/sec:

```
block size       16 B      64 B     256 B      1 KB      8 KB     16 KB
software     14116.13  19883.42  22377.09  23099.90  22646.78  22560.77
CESA           133.59    533.02   2130.05   7665.66  27340.80  33644.54
                105x       37x      10.5x        3x     1.2x      1.5x
               SLOWER    SLOWER    SLOWER    SLOWER    faster    faster
```

Per-request DMA setup and interrupt cost dominate: it doesn't break even until somewhere between
1 KB and 8 KB, which is the wrong shape for network packets and disk sectors. And its 33.6MB/s
peak is indistinguishable from this box's measured 34.3MB/s network ceiling (see the `samba`
role above), so even a perfect match would only hand the work to a different bottleneck.

**The one genuinely useful finding — a trap to avoid.** For `cbc(aes)` the kernel sees:

```
cbc(aes)  mv-cbc-aes          priority 300   <- CESA
cbc(aes)  cbc(ecb(aes-lib))   priority 100   <- software
```

Selection is by priority, so **any in-kernel `cbc(aes)` user gets CESA automatically**. dm-crypt
operates on 512B or 4K sectors — squarely in the range where CESA is 3-10x *slower* — so setting
up an encrypted volume with `aes-cbc-*` here would produce a silent performance regression
*caused by the accelerator*. If an encrypted volume is ever wanted on this box, prefer
`aes-xts-plain64` (XTS isn't in CESA's list, so software handles it) or benchmark first.

## Usage

### Required variables (`host_vars/iomega.yml`)

`ansible/host_vars/` is gitignored, so a fresh clone doesn't have it. Create
`ansible/host_vars/iomega.yml` (the filename must match the inventory hostname) before the
first run:

```yaml
# ntfy.sh topic for RAID/SMART/sensor alerts. Treat it like a password: pick something long
# and unguessable — anyone who knows it can read your alerts or post fake ones.
monitoring_ntfy_topic: <long-random-topic-name>

# Samba accounts (each also gets a nologin system user and a private home share).
samba_users:
  - name: admin
    password: <samba-password>
```

Both roles `assert` against their `CHANGE_ME_SEE_HOST_VARS` placeholders and stop rather than
deploy with them.

Values that are specific to this one box and need changing for any other unit live in tracked
files instead:

- `inventory.ini` — hostname `iomega`, SSH user `admin`, key `~/.ssh/id_ed25519_iomega`.
- `roles/monitoring/defaults/main.yml` — `monitoring_smart_devices`, the two disks'
  `/dev/disk/by-id` paths (serial-keyed). smartd would watch nonexistent devices on any other
  hardware. `ls /dev/disk/by-id/ata-*` on the box lists the right ones.
- `roles/security/defaults/main.yml` — `security_admin_user` (`admin`).

### Running

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
- **Adding the second disk to the RAID arrays** (`mdadm --add`). A live, carefully-sequenced,
  by-hand task, not a routine idempotent playbook task. **Stage 1 done 2026-09-11**: `md0`/`/boot`
  and `md1`/`/` are now real 2-disk mirrors (`sdb`'s original 1TB disk + a repartitioned-to-match
  `sda`, 4TB) — partitioned via `sfdisk` to exactly replicate `sdb`'s MBR layout (`sda` was wiped
  from its prior GPT/exFAT state first; its earlier contents were a completed, already-sorted
  PhotoRec recovery, not live data), `mdadm --add`ed, resynced, then verified by an actual reboot
  with both disks attached — see `docs/boot.md`'s "Verified 2026-09-11" note. **`md2`/`/srv/data`
  (912GB) is deliberately still degraded, single-disk** — that mirror + its own reboot-survival
  test is a separate, later pass (a multi-hour resync, staged intentionally after the smaller
  boot/root test proved the two-disk boot chain safe). Also note: this device's U-Boot can't read
  GPT and MBR caps out around ~2TiB regardless of physical disk size — see `docs/boot.md` for the
  full detail — so even once `md2` is mirrored onto the 4TB disk, usable capacity there tops out
  around ~1.95TiB, not the full drive.
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
