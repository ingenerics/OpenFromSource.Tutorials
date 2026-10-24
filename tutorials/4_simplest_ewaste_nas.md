# Tutorial 4 — An offline NAS installed from a USB stick

◂ [Tutorial 3 — Hyprland from backports, and the version-drift trap](3_hyprland_version_drift.md) · [All tutorials](../README.md) · [Tutorial 5 — The full base set, straight from the installer DVD](5_ensure_base_packages.md) ▸

They say complexity brings technical debt. Let's flip it: the simplest NAS possible — a Debian box, one NFS export, a firewall — set up with no internet, using only the local cache on a USB stick. This chapter adds a third system to the cast: the NAS box itself, bootstrapped *live off the stick*, then hardened and serving its storage over NFS to the rest of the LAN.

> **Good to know** — just to be sure, this is merely meant as a "baby's first live share" tutorial. There a feature complete (open source) alternatives that will also run on a potato. One thing you'll likely run into is that copying many small files over the network will take ages. At least without proper setup or workarounds (like creating tarballs).

## At a glance

| | |
|---|---|
| **You'll end up with** | A NAS exporting `/srv` over NFSv4 to your LAN, firewalled with UFW, set up entirely offline |
| **You need** | Tutorials 1–3 done · NAS box with Debian trixie installed (a thumb drive works) · cache on a USB stick · a client on the LAN |
| **Time** | ~25 min active, ~5 min waiting |
| **Difficulty** | ★★☆ |
| **New concepts** | live-mount bootstrap, NFS, exports, UFW |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | fills the cache, packs a handoff for the NAS |
| **NAS box** | e-waste (offline) | `vit@e-waste ~$` | the NAS — bootstrapped off the stick, exports NFS |
| **Client** | ex-box (offline) | `op@ex-box ~$` | mounts the share, proves it works |

> **Say what? — "NFS":** network file system — a folder on one machine that other machines on the LAN can mount as a plain directory. No internet involved, just your own network.

> Stick mount points are normalized here (the recorded run used a dated backup path). This tutorial assumes the cache stick mounts at `/mnt/offline-cache`.

## Before you start

- [ ] Tutorials 1–3 complete — cache on bis, `diff` cleared
- [ ] NAS box is a fresh Debian trixie install, with SSH server and standard system utilities
- [ ] Cache USB stick formatted ext4 or btrfs (symlinks required, FAT will not work)
- [ ] A client on the LAN (ex-box) to mount the share

## Milestone 1 — Pack a handoff for the NAS

### Step 1.1 — Add the NFS set

The NAS is a server now, so it needs three things: the NFS server, its client tooling, and a firewall.

```txt
nfs-kernel-server
nfs-common
ufw
```

```bash
vit@bis OpenFromSource$ pm add --file /tmp/nfs_server.txt
```

You should get something like:

```console
vit@bis OpenFromSource$ pm add --file /tmp/nfs_server.txt
INFO:package_mgmt.lib.service.resolver:Indexes loaded: 73148 binary packages, 0 source packages (2 sources configured)
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libgcc-s1 -> libc6
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: openssl-provider-legacy -> libssl3t64
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: dmsetup -> libdevmapper1.02.1
Resolved 3/3
Packages requested:  3
Packages resolved:   3
Errors:              0
```

### Step 1.2 — Verify and download

```bash
vit@bis OpenFromSource$ pm list --deps nfs-kernel-server
$ pm verify
```

Then download — the whole NFS set is under a megabyte:

```bash
vit@bis OpenFromSource$ pm download
```

```console
vit@bis OpenFromSource$ pm download
Elapsed: 00:00:06
[████████████████████████████████████████] 10/10 files (1023.1 KiB/1023.1 KiB)

Binary phase summary
────────────────────
Files downloaded:  10
Bytes:             1023.1 KiB
Files verified:    10
Failed:            0
Elapsed:           00:00:06
…
Total: 10 files downloaded, 10 verified, 0 failed (00:00:06)
```

### Step 1.3 — Register the NAS and pack its handoff

The NAS is a new consumer, so it gets its own **moniker**:

```bash
vit@bis OpenFromSource$ pm systems add e-waste
$ pm handoff --system e-waste
```

You should get something like:

```console
INFO:package_mgmt.lib.service.handoff:handoff generated: 156 packages across 2 release(s) (trixie, trixie-backports)
Handoff ready: /var/cache/package_mgmt/binary/handoff
Release:        trixie, trixie-backports
Components:     trixie: main, contrib, non-free; trixie-backports: main
Packages:       156 (trixie/main: 156, trixie/contrib: 0, trixie/non-free: 0, trixie-backports/main: 0)
Signed:         yes (key offline-cache.gpg)
Bootstrap:      /var/cache/package_mgmt/binary/handoff/bootstrap.sh
Snapshot:       156 binaries marked for 'e-waste'
```

Copy the cache directory onto the stick and take it to the NAS.

## Milestone 2 — Bootstrap the NAS live off the stick

In Tutorial 1 the operator copied the cache onto the machine first. A NAS is often different: it boots from a small thumb drive and shouldn't waste space on a cache copy. The good news — `bootstrap.sh` accepts a cache directory argument (it defaults to the parent of the script's own directory), so you can run it **straight off the USB** and only mount the cache for as long as you need it.

> **Say what? — "live-mount bootstrap":** running `bootstrap.sh` directly from the mounted stick instead of copying the cache onto the machine. Apt's sources point at the stick's handoff; once the packages are installed the stick can go.

### Step 2.1 — Mount the stick at a stable mountpoint

The sources entry bakes in the absolute path used at bootstrap time, so mount the stick (update the paths below) at a fixed place and leave it there until you're done:

```bash
vit@e-waste ~$ sudo mount /dev/sdb1 /mnt/offline-cache
```

### Step 2.2 — Run the bootstrap straight off the stick

```bash
vit@e-waste ~$ sudo /mnt/offline-cache/package_mgmt/binary/handoff/bootstrap.sh
```

<details><summary>Full bootstrap log</summary>

```console
vit@e-waste ~$ sudo /mnt/offline-cache/package_mgmt/binary/handoff/bootstrap.sh
Ign:1 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie InRelease
Ign:2 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports InRelease
Get:3 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie Release [2,991 B]
Get:4 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports Release [1,122 B]
Get:3 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie Release [2,991 B]
Get:4 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports Release [1,122 B]
Get:5 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie Release.gpg [488 B]
Get:5 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie Release.gpg [488 B]
Get:6 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports Release.gpg [488 B]
Get:6 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports Release.gpg [488 B]
Get:7 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie/main amd64 Packages [107 kB]
Get:8 file:/mnt/offline-cache/package_mgmt/binary/handoff trixie-backports/main amd64 Packages [6,594 B]
181 packages can be upgraded. Run 'apt list --upgradable' to see them.
Bootstrap: configuring apt from /mnt/offline-cache/package_mgmt/binary/handoff (releases: trixie, trixie-backports)
Source added: /etc/apt/sources.list.d/openfromsource-cache.list
apt update: OK
```

</details>

> **Watch out** — three rules for the live-mount trick: (1) the stick must be **ext4/btrfs**, because the handoff links `pool → ../pool` and the bootstrap hard-fails on media without symlinks (FAT/exFAT); (2) keep the **mountpoint stable** — the sources entry stores the absolute path, so if you remount elsewhere, re-run the bootstrap to rewrite the entry or apt breaks; (3) **keep it mounted until the installs finish** — apt reads the pool through that symlink during install.

> **Good to know** — the bootstrap is idempotent: re-running it is safe. And when you're truly done with the cache, `bootstrap.sh --remove` tears the apt entry back down.

### Step 2.3 — Upgrade and prove the box is served

```bash
vit@e-waste ~$ sudo apt update
$ sudo apt upgrade
```

The installer's media is older than the cache, so expect a pile of upgrades — all served from the stick. Unlike the kernel chapter, there's nothing special to pin here: the base kernel is fine for a simple share, so skip backports.

Now the final test:

```bash
vit@e-waste ~$ sudo apt install vifm neovim kitty
```
Output

```console
vit@e-waste ~$ sudo apt install vifm neovim kitty
Installing dependencies:
  fontconfig-config  libcairo2  libluajit-5.1-2  …  libxkbcommon-x11-0  …
…
Summary:
  Upgrading: 0, Installing: 36, Removing: 0, Not Upgrading: 186
  Download size: 0 B / 24.1 MB
  Space needed: 0 B / 4,509 MB available

Continue? [Y/n]
```

**Download size: 0 B** — 36 packages off the stick, nothing from the internet. 

Keep the USB drive mounted.

## Milestone 3 — Export a share over NFS

Now the NAS earns its name. First the server, then hardening, then a firewall, then the export.

### Step 3.1 — Install the NFS server

```bash
vit@e-waste ~$ sudo apt install nfs-kernel-server
```

### Step 3.2 — Harden NFS to v4 only

NFSv2 and NFSv3 are older, unauthenticated protocols nobody needs on a home LAN. Disable them. In `/etc/default/nfs-kernel-server`, add:

```txt
# Disable NFSv2 and NFSv3 requests
RPCNFSDOPTS="-N 2 -N 3"
RPCMOUNTDOPTS="--manage-gids -N 2 -N 3"
```

Then restart the server:

```bash
vit@e-waste ~$ sudo systemctl restart nfs-server
```

> **Good to know** — `--manage-gids` makes the server resolve group memberships for clients that can't, which keeps group permissions working across the mount. It's on by default in modern setups; `-N 2 -N 3` is the security-relevant part.

### Step 3.3 — Firewall with UFW

UFW came with the cache set. Add rules to only allow the LAN access through SSH and NFS, then switch it on:

```bash
vit@e-waste ~$ sudo apt install ufw
$ sudo ufw allow from 192.168.0.0/16 to any port ssh
$ sudo ufw enable
$ sudo ufw allow from 192.168.0.0/16 to any port nfs
$ sudo ufw reload
$ sudo ufw status
```

```console
vit@e-waste ~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       192.168.0.0/16               
2049                       ALLOW       192.168.0.0/16
```

> **Say what? — "UFW":** the friendly front-end over iptables/nftables. `allow from <net> to any port nfs` opens the NFS port (2049) for your LAN only — everything else stays closed.

> **Good to know** — this tutorial uses the `192.168.0.0/16` LAN. If your network is `172.16.0.0/24` or similar, substitute it in both `ufw allow` lines.

### Step 3.4 — Export the storage

Find the data disk with `lsblk`:

```bash
vit@e-waste ~$ lsblk
```

<details><summary>Full lsblk output</summary>

```console
vit@e-waste ~$ lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda            8:0    0  3.6T  0 disk 
└─sda1         8:1    0  3.6T  0 part /srv/bcup
sdb            8:16   0  3.6T  0 disk 
└─sdb1         8:17   0  3.6T  0 part /srv/local
sdd            8:48   1 28.7G  0 disk 
├─sdd1         8:49   1  976M  0 part /boot/efi
├─sdd2         8:50   1  5.7G  0 part /
├─sdd3         8:51   1  2.3G  0 part /var
├─sdd4         8:52   1  977M  0 part [SWAP]
└─sdd5         8:53   1 18.8G  0 part /srv
mmcblk0      179:0    0  3.7G  0 disk 
├─mmcblk0p1  179:1    0   16M  0 part 
├─mmcblk0p2  179:2    0  224M  0 part 
├─mmcblk0p3  179:3    0  224M  0 part 
├─mmcblk0p5  179:5    0    8M  0 part 
└─mmcblk0p6  179:6    0    8M  0 part 
mmcblk0boot0 179:256  0    2M  1 disk 
mmcblk0boot1 179:512  0    2M  1 disk 
```

</details>

Here `/dev/sdd` is the Trixie boot drive and `/dev/sda`/`/dev/sdb` are the actual storage disks, already mounted at `/srv/bcup` and `/srv/local`. We'll export the whole `/srv` tree so both stay reachable over the LAN.

Give the export to the nobody/nogroup user (that's what NFS maps anonymous requests to), then declare the export in `/etc/exports`:

```bash
vit@e-waste ~$ sudo chown -R nobody:nogroup /srv
```

```txt
/srv    192.168.0.0/255.255.0.0(rw,sync,fsid=0,crossmnt,no_subtree_check)
```

> **Say what? — "exports":** `/etc/exports` — the file that says which directories the NFS server shares and with whom. One line per share: path, allowed network, options.

Restart and list what the server is exporting:

```bash
vit@e-waste ~$ sudo systemctl restart nfs-server
$ sudo showmount -e
```

```console
vit@e-waste ~$ sudo showmount -e
Export list for e-waste:
/srv 192.168.0.0/255.255.0.0
```

> **Watch out** — this is a deliberately loose config: `rw` for the whole LAN, no auth, meaning any sudo user on an allowed machine can write to the NAS. It's not best practice; it's the "no internet, get me going" config. Proper Kerberos/NFSv4 security is left as an exercise.

## Milestone 4 — Mount it from the client

Back on ex-box, the client side needs the NFS client tools, then the share mounts like any directory:

```bash
op@ex-box ~$ sudo apt install nfs-common
$ sudo mount.nfs4 192.168.60.70:/ /mnt
```

`192.168.60.70` is the NAS's LAN address; `/srv` is exported as the root (`fsid=0`), so mounting `:/` brings in the whole `/srv` tree under `/mnt`.

## Checkpoint — prove the NAS serves

From the NAS side, the export is visible:

```console
vit@e-waste ~$ sudo showmount -e
Export list for e-waste:
/srv 192.168.0.0/255.255.0.0
```

From the client, the data is there:

```bash
op@ex-box ~$ ls /mnt
```

You should see the NAS's storage — `/srv/bcup` and `/srv/local` — as plain directories on the offline client. A whole NAS, built and served with zero bytes over the internet.

> **Good to know** — NFSv4 (`mount.nfs4`) locks clients to the server for the life of the mount, so an unmounted NAS can hang a client that had it mounted. If the NAS is going to sleep, unmount the client first.

> **Good to know** — there's something called the 3-2-1 backup rule. There's definitely a place (and time) for repurposing e-waste as an extra backup. It doesn't need to be on 24-7. 

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `bootstrap.sh` fails on the stick | Filesystem without symlinks (FAT/exFAT) | Reformat ext4/btrfs and re-copy the cache |
| `apt` can't find packages after remounting the stick | The sources entry bakes in the old mountpoint | Re-mount at the same path, or re-run `bootstrap.sh` |
| Install fails partway | The stick got unmounted mid-install | Re-mount and run `apt install` again; the cache is only read during install |
| `showmount -e` shows nothing | NFS server didn't pick up the export | Check `/etc/exports` syntax, then `sudo systemctl restart nfs-server` |
| Client mount hangs | NFSv4 client waiting on a server that's gone | `umount -l /mnt`, and unmount before the NAS goes to sleep |
| UFW blocks the share | Only SSH was allowed | Re-run `ufw allow from <lan> to any port nfs` and `ufw reload` |

## Decoder ring

- **NFS** — network file system; a folder on one machine mounted as a directory on another over the LAN
- **live-mount bootstrap** — running `bootstrap.sh` directly off the mounted USB cache instead of copying the cache onto the machine
- **exports** — `/etc/exports`; the file listing which directories the NFS server shares and with whom
- **UFW** — the friendly front-end over iptables/nftables for firewall rules

## Next

[Tutorial 5 — The full base set, straight from the installer DVD](5_ensure_base_packages.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*
