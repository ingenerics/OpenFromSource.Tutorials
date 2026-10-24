# Tutorial 2 — A newer kernel from backports

◂ [Tutorial 1 — Creating an offline cache](1_creating_an_offline_cache.md) · [All tutorials](../README.md) · [Tutorial 3 — Hyprland from backports, and the version-drift trap](3_hyprland_version_drift.md) ▸

The offline box works, but the kernel on the installer DVD is already quite old: trixie ships 6.12.107, and our plans envision the 7.1.8 kernel that only exists in backports. This chapter pulls it through the same cache pipeline you already know — add, download, sync, apply, install — and proves the box is now booting the newer kernel, still with **Download size: 0 B**.

> **Say what? — "backports":** packages rebuilt from the *next* Debian release for the current one. You stay on trixie and still get a newer version of a package — here, a 7.1.8 kernel that trixie itself doesn't carry.

> **Watch out** — the backports repo might sound like a good idea for leaf dependencies like the linux kernel. However, in the next tutorial we'll pull Hyprland from backports and Hyprland brings quite a bit of dependencies that are shared with lots of other binaries. This introduces version drift and can quickly amount into a broken system.

## At a glance

| | |
|---|---|
| **You'll end up with** | ex-box running kernel 7.1.8 from trixie-backports, installed entirely offline |
| **You need** | Tutorial 1 done (cache + handoff live on ex-box) · USB stick (ext4/btrfs) · admin + operator access |
| **Time** | ~15 min active, ~15 min waiting (the ~160 MB kernel modules download) |
| **Difficulty** | ★★☆ |
| **New concepts** | backports, `--source`, metapackage, sync/diff |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | pins the backported kernel, ships a diff |
| **Operator** | ex-box (offline) | `op@ex-box ~$` | applies the diff, installs, reboots |

## Before you start

- [ ] Tutorial 1 complete — cache initialized, `ex-box` moniker created, ex-box bootstrapped
- [ ] Create `project.local.toml` at the root directory of the project:

      ```toml
      [[archive.source]]
      mirror = "http://deb.debian.org/debian"
      release = "trixie"
      components = ["main", "contrib", "non-free"]

      [[archive.source]]
      mirror = "http://deb.debian.org/debian"
      release = "trixie-backports"
      components = ["main"]
      ```

      Only these keys are overridden; the rest comes from `project.defaults.toml`.

- [ ] USB stick at hand for the diff

## Milestone 1 — Pin the backported kernel on the admin side

The kernel is a special customer: you want a *specific* backports build, not "whatever's newest whenever you look." The way to get that is to pin the version in the package list, then tell the resolver to prefer backports for the whole batch.

### Step 1.1 — Write the kernel manifest

Two lines: the image metapackage and the headers, both pinned to the backports build `7.1.8-1~bpo13+1`.

```txt
# backports kernel image + headers, pinned to the backports build
linux-image-amd64=7.1.8-1~bpo13+1
linux-headers-amd64=7.1.8-1~bpo13+1
```

> **Say what? — "metapackage":** `linux-image-amd64` is a tiny package that does nothing but depend on the real kernel package and track the newest one. Pinning the metapackage's version pins the kernel, without you ever naming `linux-image-7.1.8+deb13-amd64` yourself.

> **Good to know** — the available kernel versions changes over time. If the versions above can no longer be found, you can drop the pin (`=7.1.8-1~bpo13+1`).

### Step 1.2 — Add it, from backports

```bash
vit@bis OpenFromSource$ pm add --file /tmp/kernel.txt --source trixie-backports
```

> **Say what? — "--source":** overrides which archive the resolver searches *first*. Here it looks at `trixie-backports` before anything else; anything the backports kernel needs that isn't in backports falls back to trixie automatically.

You should get something like:

```console
vit@bis OpenFromSource$ pm add --file /tmp/kernel.txt --source trixie-backports
INFO:package_mgmt.lib.service.resolver:Indexes loaded: 73148 binary packages, 0 source packages (2 sources configured)
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libgcc-s1 -> libc6
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: openssl-provider-legacy -> libssl3t64
Resolved 2/2
Packages requested:  2
Packages resolved:   2
Errors:              0
```

> **It's fine** — the `WARNING` about `Packages.gz` "not listed in the signed release" for backports. Debian publishes the backports index compressed as `.xz`; `package_mgmt` noticed, fetched the `.xz` instead, and carried on.

### Step 1.3 — Verify and download

```bash
vit@bis OpenFromSource$ pm list --deps linux-image-amd64
$ pm verify
```

Both should end with a `COMPLETE` status. Then download — and don't panic when the progress bar parks itself:

```bash
vit@bis OpenFromSource$ pm download
```
Output

```console
Elapsed: 00:13:17
[████████████████████████████████████████] 56/56 files (235.9 MiB/235.9 MiB)

Binary phase summary
────────────────────
Files downloaded:  56
Bytes:             235.9 MiB
Files verified:    56
Failed:            0
Elapsed:           00:13:17
…
Total: 56 files downloaded, 56 verified, 0 failed (00:13:17)
```

56 files, 235.9 MiB, every one verified.

## Milestone 2 — Sync a diff, not a new handoff

Tutorial 1 shipped the whole cache as a handoff. That would be wasteful now — ex-box already has the base set. Instead, `pm sync` compares what ex-box has against the cache and emits only the difference: a **diff**, small enough for the stick.

> **Say what? — "diff":** in this tool's terms, a signed delta for one consumer — the new packages only, with an `apply.sh` to merge them into the operator's cache. The next sync builds on the same base, so every trip is small.

```bash
vit@bis OpenFromSource$ pm sync --system ex-box
```

```console
vit@bis OpenFromSource$ pm sync --system ex-box
INFO:package_mgmt.lib.service.handoff:diff generated: 56 new across 2 release(s) (trixie, trixie-backports), 156 delivered in the base set
Sync ready:      /var/cache/package_mgmt/binary/diff
Release:         trixie, trixie-backports
Components:      trixie: main, contrib, non-free; trixie-backports: main
New packages:    56 (trixie/main: 44, trixie-backports/main: 12)
Total offered:   212 (prior 156 + new 56)
Signed:          yes (key offline-cache.gpg, same as handoff)
Apply:           /var/cache/package_mgmt/binary/diff/apply.sh
Snapshot:        56 binaries marked for 'ex-box'
```

Copy the `diff` directory onto the stick, same as you did the cache in Tutorial 1. Only the new 56 packages travel this time.

## Milestone 3 — Apply the diff and install the kernel

Back on the offline box. The handoff is already live from Tutorial 1; now the diff extends it.

### Step 3.1 — Copy the diff into place

Mount the stick, then merge the diff into the binary cache:

```bash
op@ex-box cache$ sudo cp -r diff /var/cache/package_mgmt/binary/
```

### Step 3.2 — Run the apply script

```bash
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
```

```console
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
Apply: merging diff into /var/cache/package_mgmt/binary (releases: trixie, trixie-backports)
New files:       56
Offered after:   212 packages (56 new)
Handoff indexes: replaced for trixie trixie-backports
apt update: OK
```

`apt update: OK` — the handoff index now advertises the kernel, signed with the same key apt already trusts.

### Step 3.3 — Install the kernel

```bash
op@ex-box ~$ sudo apt install linux-image-amd64 linux-headers-amd64
```

You should see the plan — note that apt treats the backports kernel as a newer version, no special flags needed:

```console
op@ex-box ~$ sudo apt install linux-image-amd64 linux-headers-amd64
Upgrading:
  libelf1t64  linux-image-amd64

Installing:
  linux-headers-amd64

Installing dependencies:
  binutils-common  …  linux-modules-7.1.8+deb13-amd64  …

Summary:
  Upgrading: 2, Installing: 34, Removing: 0, Not Upgrading: 174
  Download size: 0 B / 239 MB
  Freed space: 1,271 kB

Continue? [Y/n]
```

**Download size: 0 B** again — the whole 239 MB comes from the cache. The details below show the tail of the install: new initramfs, new GRUB entry, and the old kernel kept as a fallback.

<details><summary>Full kernel install log</summary>

```console
Get:1 file:/var/cache/package_mgmt/binary/handoff trixie-backports/main amd64 linux-base-7.1.8+deb13-amd64 amd64 7.1.8-1~bpo13+1 [1,118 kB]
Get:2 file:/var/cache/package_mgmt/binary/handoff trixie-backports/main amd64 linux-binary-7.1.8+deb13-amd64 amd64 7.1.8-1~bpo13+1 [13.8 MB]
Get:3 file:/var/cache/package_mgmt/binary/handoff trixie-backports/main amd64 linux-modules-7.1.8+deb13-amd64 amd64 7.1.8-1~bpo13+1 [159 MB]
Get:4 file:/var/cache/package_mgmt/binary/handoff trixie-backports/main amd64 linux-image-7.1.8+deb13-amd64 amd64 7.1.8-1~bpo13+1 [1,896 B]
<SNIP — 31 more Get: lines>
Unpacking linux-image-amd64 (7.1.8-1~bpo13+1) over (6.12.107-1) ...
…
Setting up linux-image-7.1.8+deb13-amd64 (7.1.8-1~bpo13+1) ...
I: /vmlinuz is now a symlink to boot/vmlinuz-7.1.8+deb13-amd64
I: /initrd.img is now a symlink to boot/initrd.img-7.1.8+deb13-amd64
/etc/kernel/postinst.d/initramfs-tools:
update-initramfs: Generating /boot/initrd.img-7.1.8+deb13-amd64
/etc/kernel/postinst.d/zz-update-grub:
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.1.8+deb13-amd64
Found initrd image: /boot/initrd.img-7.1.8+deb13-amd64
Found linux image: /boot/vmlinuz-6.12.107+deb13-amd64
Found initrd image: /boot/initrd.img-6.12.107+deb13-amd64
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
Setting up linux-image-amd64 (7.1.8-1~bpo13+1) ...
…
Setting up linux-headers-amd64 (7.1.8-1~bpo13+1) ...
Processing triggers for man-db (2.13.1-1) ...
Processing triggers for libc-bin (2.41-12+deb13u4) ...
```

> **It's fine** — the `os-prober` warning. GRUB keeps the old 6.12.107 entry as a fallback; nothing else was detected to boot.

</details>

### Step 3.4 — Reboot

```bash
op@ex-box ~$ sudo reboot
```

## Checkpoint — prove the running kernel is the backports one

After the box comes back up, the running kernel should be 7.1.8:

```console
op@ex-box ~$ cat /proc/version
Linux version 7.1.8+deb13-amd64 (debian-kernel@lists.debian.org) (x86_64-linux-gnu-gcc-14 (Debian 14.2.0-19) 14.2.0, GNU ld (GNU Binutils for Debian) 2.44) #1 SMP PREEMPT_DYNAMIC Debian 7.1.8-1~bpo13+1 (2026-08-16)
```

`7.1.8+deb13-amd64` with `~bpo13+1` in the build, that's the backports kernel running on our offline box.

## Milestone 4 — Clear the diff

The diff has served its purpose. Delete it — and make a point of doing this after every apply, because the next sync needs that same location.

```bash
op@ex-box ~$ sudo rm -rf /var/cache/package_mgmt/binary/diff
```

> **Watch out** — never delete the `handoff` directory. It's the live apt source on the operator machine: `bootstrap.sh` wrote the sources list pointing straight at it. The diff is disposable; the handoff is the repository.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `Errors:` is not 0 on `add` | The pinned version isn't in backports anymore | Check `packages.debian.org/trixie-backports/linux-image-amd64` and update the pin |
| `apply.sh` reports a stale handoff | The diff is out of step with the cache | Re-run `pm sync --system ex-box` and re-copy the fresh `diff` |
| `apt install` wants to download | `diff` landed in the wrong place, or `handoff/` was touched | Confirm `/var/cache/package_mgmt/binary/handoff` is intact, re-run apply |
| Still booting 6.12 after reboot | GRUB picked the old entry | Choose the 7.1.8 entry in the GRUB menu, or `sudo update-grub` |
| Next sync fails with "location in use" | Old `diff` folder still present | `sudo rm -rf /var/cache/package_mgmt/binary/diff` (it's safe) |

## Decoder ring

- **backports** — packages rebuilt from the next Debian release for the current one; newer versions, still on trixie
- **`--source`** — override which archive the resolver searches first; other sources are the fallback
- **metapackage** — a tiny package that only depends on the real package and tracks the newest version
- **diff** — a signed delta for one consumer: only the new packages, with an `apply.sh`

## Next

[Tutorial 3 — Hyprland from backports, and the version-drift trap](3_hyprland_version_drift.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*