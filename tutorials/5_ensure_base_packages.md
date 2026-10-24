# Tutorial 5 — Ensuring the full base set from the installer DVD

◂ [Tutorial 4 — An offline NAS installed from a USB stick](4_simplest_ewaste_nas.md) · [All tutorials](README.md) · [Tutorial 6 — Building Hyprland from source](6_hyprland_from_source.md) ▸

Everything so far ran on hand-picked package lists. This is the upgrade path to "turtles all the way down": a cache holding the **entire Debian base** — every binary *and* every source, with every build tool — so any machine, any package, can be rebuilt offline. The installer DVD is the key: its index seeds the whole base set. This chapter builds that cache: 14,856 binaries and 26,527 sources, 34 GiB, every file verified.

> **It's fine** — Captain Obvious chiming in; if you have no use for these sources and build tools (and are not a compulsive data hoarder), there's nothing wrong with sitting this one out (and just reading along). 

## At a glance

| | |
|---|---|
| **You'll end up with** | A cache holding the full Debian trixie base — 14,856 binaries + 26,527 sources, all verified and rebuildable offline |
| **You need** | Admin box (bis) with internet · Debian 13.7 amd64 **DVD-1** ISO (~3.8 GB) · ~40 GiB free disk · ~1.5 h patience |
| **Time** | ~30 min active, ~1.5 h waiting (downloads) |
| **Difficulty** | ★★☆ |
| **New concepts** | ensure-base, provides |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | builds the full base set |

No operator this chapter — this is admin-side work, preparing the cache that later chapters (and the whole alpha release) build on.

## Before you start

- [ ] Tutorial 1 complete — tool installed (`uv` + venv, `pm` alias in `~/.bashrc`)
- [ ] Admin box with internet and ~40 GiB free disk
- [ ] Debian 13.7 amd64 DVD-1 ISO downloaded (`debian-13.7.0-amd64-DVD-1.iso`, ~3.8 GB)
- [ ] You can become root

> **Watch out** — `ensure-base` only runs on an **empty** cache. This chapter builds a fresh, separate base cache; if you kept the Tutorials 1–4 cache on this box, back it up or point `base_dir` elsewhere before wiping.

## Milestone 1 — Prepare a fresh, graybeard cache

The tool is installed and `pm` is in your `~/.bashrc` (Tutorial 1). This chapter builds a separate base cache with the graybeard profile — and `ensure-base` only runs on an **empty** cache, so this one starts from scratch.

### Step 1.1 — Create the cache directory and drop in the ISO

```bash
$ export CACHEPATH="/var/cache/package_mgmt"
$ sudo mkdir $CACHEPATH
$ sudo chown -R $USER:$USER $CACHEPATH
```

`ensure-base` reads the installer image from the cache's `iso/` directory. Copy the DVD there:

```bash
vit@bis OpenFromSource$ ll /var/cache/package_mgmt/iso/
```
Output

```console
vit@bis OpenFromSource$ ll /var/cache/package_mgmt/iso/
total 3.8G
drwxr-xr-x 2 vit vit 4.0K Oct 20 20:09 .
drwxr-xr-x 3 vit vit 4.0K Oct 20 20:09 ..
-rw-r--r-- 1 vit vit 3.8G Oct 20 20:09 debian-13.7.0-amd64-DVD-1.iso
```

> **Say what? — "ensure-base":** the command that ingests the installer DVD's index into the cache, registering every package on the disc as the base set. The image must be the first DVD of the release (DVD-1), and exactly one image may be present.

### Step 1.2 — Set the graybeard profile

The graybeard profile fetches **everything**: every binary's source and every source's build tools, at unlimited depth. It's the intended `ensure-base` configuration. Merge its sections into `project.local.toml`:

```toml
[[archive.source]]
mirror = "http://deb.debian.org/debian"
release = "trixie"
components = ["main", "contrib", "non-free", "non-free-firmware"]

[fetch]
# graybeard: fetch everything — sources and build tools for every binary.
source = true

[resolution]
# Depth convention: -1 = unlimited, 0 = none, positive N = up to N levels.
# -1 (unlimited) = full recursion — every binary gets its source and build tools.
max_source_dep_depth = -1
```

> **Say what? — "graybeard profile":** the graybeard preset from Tutorial 1 — sources and build tools for the whole closure, no depth cap. The intended `ensure-base` configuration. The cost is scale: the cache grows past 30 GB.

## Milestone 2 — Initialize the graybeard cache

### Step 2.1 — Initialize

From the project root (may take a minute):

```bash
vit@bis OpenFromSource$ pm init
```

<details><summary>Full init log</summary>

```console
vit@bis OpenFromSource$ pm init
INFO:package_mgmt.lib.service.initializer:Binary cache /var/cache/package_mgmt/binary (was created)
INFO:package_mgmt.lib.service.initializer:Source cache /var/cache/package_mgmt/source (was created)
INFO:package_mgmt.lib.service.db_init:Database created at /var/cache/package_mgmt/package_mgmt.db (schema v1)
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/main/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/main/source/Sources.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/contrib/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/contrib/source/Sources.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/non-free/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/non-free/source/Sources.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/non-free-firmware/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/non-free-firmware/source/Sources.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.gz
WARNING:package_mgmt.cli.commands.init:Failed to fetch https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.gz: index main/binary-amd64/Packages.gz is not listed in the signed release at https://deb.debian.org/debian/dists/trixie-backports
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.xz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/source/Sources.gz
WARNING:package_mgmt.cli.commands.init:Failed to fetch https://deb.debian.org/debian/dists/trixie-backports/main/source/Sources.gz: index main/source/Sources.gz is not listed in the signed release at https://deb.debian.org/debian/dists/trixie-backports
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/source/Sources.xz
INFO:package_mgmt.lib.service.initializer:Populated 69989 provides entries into database
Base directory:    /var/cache/package_mgmt
Binary cache:      /var/cache/package_mgmt/binary
Source cache:      /var/cache/package_mgmt/source
Database:          /var/cache/package_mgmt/package_mgmt.db
Schema version:    1
Default source:    trixie/main
Archive sources:
  [1] trixie/main — 68825 binaries, 37643 sources
  [2] trixie/contrib — 303 binaries, 159 sources
  [3] trixie/non-free — 741 binaries, 262 sources
  [4] trixie/non-free-firmware — 44 binaries, 14 sources
  [5] trixie-backports/main — 3356 binaries, 942 sources
Provides entries:  69989
Status:            initialized
```

> **It's fine** — the backports `Failed to fetch` warnings are the same `.xz`-instead-of-`.gz` behavior as Tutorial 1, now also hitting the *source* indexes (`Sources.gz`). Notice this init also pulls those source indexes and adds `non-free-firmware` — the graybeard source set.

</details>

## Milestone 3 — Register the base set from the DVD

### Step 3.1 — Ingest the image

```bash
vit@bis OpenFromSource$ pm ensure-base
```

<details><summary>Full ensure-base log (trimmed)</summary>

```console
vit@bis OpenFromSource$ pm ensure-base
INFO:package_mgmt.lib.service.db_init:Database schema validated at /var/cache/package_mgmt/package_mgmt.db (v1)
INFO:package_mgmt.lib.service.initializer:Populated 0 provides entries into database
INFO:package_mgmt.lib.service.resolver:Indexes loaded: 73269 binary packages, 39020 source packages (2 sources configured)
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libgcc-s1 -> libc6
Resolved 1/4448WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: openssl-provider-legacy -> libssl3t64
Resolved 24/4448WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libheif-plugin-dav1d -> libheif1
<SNIP — the resolver walks all 4,448 names; circular-dependency warnings interleave with Resolved X/4448 progress>
Resolved 4448/4448WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: emacs-el -> emacs-common
…
INFO:package_mgmt.lib.service.ensure_base:ensure-base registered 4448 binaries, 0 provides from /var/cache/package_mgmt/iso/debian-13.7.0-amd64-DVD-1.iso (trixie); 0 index/image mismatches; 4447/4448 packages resolved, 23 errors

ISO image:        debian-13.7.0-amd64-DVD-1.iso (trixie: contrib, main, non-free-firmware)
Binary packages:  4448
Provides entries: 0
Index/image mismatches: 0
Base packages resolved: 4447/4448
Warnings:         none
Errors:           23
  - 7zip: Virtual package 'g++-12-x86-64-linux-gnu' has no providers in the index
  - 7zip: Build tool not resolved: g++-12-x86-64-linux-gnu
  - 7zip: No version of libfmt-dev satisfies constraint (<< 10)
  - 7zip: Build tool not resolved: libfmt-dev | libfmt-dev
  - 7zip: No version of openbox-dev satisfies constraint (<< 3.6.0-1)
  - 7zip: Build tool not resolved: openbox-dev | openbox-dev
  - 7zip: Build tool not resolved: openbox-dev
  - 7zip: No version of nodejs satisfies constraint (<< 12.22.5~dfsg-4~)
  - 7zip: Build tool not resolved: nodejs | nodejs
  - 7zip: Alternative group 'nodejs | nodejs' could not be resolved
<SNIP — 13 more 7zip-prefixed lines omitted>
```

</details>

4447 of 4448 packages resolved, 23 errors — and every one of those errors wears a `7zip:` prefix that is a red herring. Here's the real story:

> **It's fine** — the `7zip:` prefix is misleading; those errors are **not** from 7zip. `ensure-base` resolves all 4,448 names in one batch, and by design every build-dep error in the batch attaches to the *first* segment only — and `7zip` sorts first. The 23 lines are really five sources' build-tool gaps, and they only affect *rebuilding those sources from scratch*. Every binary in the base set resolves and downloads fine.

<details><summary>What the 23 errors actually are</summary>

| Real requirer | Missing/unsatisfiable build tool | Why it fails |
|---|---|---|
| `gcc-12` | `g++-12-x86-64-linux-gnu`, `gobjc-12-…`, `gfortran-12-…`, `gdc-12-…`, `gccgo-12-…`, `gnat-12-…`, `gm2-12-…` (7 packages) | Cross-compiler binaries whose target (`x86_64-linux-gnu`) equals the host (`amd64`). Debian doesn't build native-target cross compilers on amd64, so they're absent from the index → "no providers". They're gated behind the `!cross` build profile, which the tool doesn't evaluate. |
| `mariadb` | `libfmt-dev \| libfmt-dev` (constraint `<< 10`) | trixie ships `libfmt-dev 10.1.1+ds1-4`; the stale `<< 10` upper bound is unsatisfiable. |
| `lxappearance-obconf` | `openbox-dev \| openbox-dev` (`<< 3.6.0-1`) | trixie ships `openbox-dev 3.6.1-12+b2`; stale upper bound. |
| `obconf` | `openbox-dev` (same reason) | Same stale constraint. |
| `acorn` | `nodejs \| nodejs` (`<< 12.22.5~dfsg-4~`) | A bookworm-era upper bound; trixie ships `nodejs 20.x`. |

The `Build tool not resolved: X | X` / `Alternative group …` lines are the same entries seen twice — a Debian quirk where a build-dep lists its own name as its alternative. `verify` later attributes all these gaps to their true requirers.

</details>

### Step 3.2 — Verify the graph

```bash
vit@bis OpenFromSource$ pm verify
```

<details><summary>Full verification log (trimmed)</summary>

```console
vit@bis OpenFromSource$ pm verify
Dependency Graph Verification
─────────────────────────────
Total packages:         17611
Total binaries:         14856
Total sources:          8202
Unresolved deps:        0
Unlinked binaries:      0
Circular dependencies:  51
  - libgcc-s1 -> libc6 (circular)
  - openssl-provider-legacy -> libssl3t64 (circular)
  - dmsetup -> libdevmapper1.02.1 (circular)
<SNIP — 48 more circular pairs omitted>
Runtime Status:         COMPLETE

Build-Tool Dependencies
───────────────────────
Total build-dep links:  77498
Total build tools:      9055
Unresolved build deps:  0
Missing build tools:    12
  - g++-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gobjc-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gfortran-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gdc-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gccgo-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gnat-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - gm2-12-x86-64-linux-gnu — required by gcc-12, no record in cache
  - libfmt-dev | libfmt-dev — required by mariadb, no record in cache
  - openbox-dev | openbox-dev — required by lxappearance-obconf, no record in cache
  - openbox-dev — required by obconf, no record in cache
  - yodl | yodl — required by zsh, no record in cache
  - nodejs | nodejs — required by acorn, no record in cache
Build Status:           INCOMPLETE
```

</details>

**Runtime Status: COMPLETE.** The build-tool status is `INCOMPLETE` over exactly 12 missing tools — the same five sources from the ensure-base errors, plus `zsh`'s `yodl`. These don't block installation: every binary in the base set is present and installable. They only matter if you try to rebuild `gcc-12`, `mariadb`, `obconf`, `lxappearance-obconf`, `acorn` or `zsh` from source offline.

## Milestone 4 — Download the entire base

With all metadata acquired and verified, we can start the actual download:

```bash
vit@bis OpenFromSource$ pm download
```

On my machine it completed under an hour and a half — binaries first (12.5 GiB), then sources (21.5 GiB):

```console
vit@bis OpenFromSource$ pm download
Elapsed: 00:27:10
[████████████████████████████████████████] 14856/14856 files (12.5 GiB/12.5 GiB)
…
Elapsed: 00:45:32
[████████████████████████████████████████] 26527/26527 files (21.5 GiB/21.5 GiB)
```

## Checkpoint — the full base, verified

When the source phase finishes you have a local Debian repository containing *all* base packages — binaries and sources, build tools included:

```console
Total: 41383 files downloaded, 41383 verified, 0 failed (01:12:43)
```

41,383 files, 34 GiB, every one verified against the signed index. **Time to take a backup** — from here, this cache is the foundation the rest of the alpha release (and Tutorials 6–7) is built on.

> **Good to know** — the graybeard cache is big and slow to rebuild. Copy it somewhere safe now (a second disk, or the NAS from Tutorial 4), so an accident on the working cache doesn't cost you another hour and a half.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `pm init` fails with exit 102 from `~` | You're not at the project root; the TOML configs live there | `cd` into the repo, then retry |
| `ensure-base` refuses to run | Cache already contains registered packages (fresh-cache-only rule) | Delete the database (and re-run `init`), or point `base_dir` elsewhere |
| `ensure-base` finds no ISO | Image not in `iso/`, or more than one present | Confirm exactly one `debian-13.7.0-amd64-DVD-1.iso` is in `/var/cache/package_mgmt/iso/` |
| `Index/image mismatches: N` | Damaged image — index lists files the disc lacks | Re-download the ISO and re-ingest from a fresh cache |
| `download` fails partway | Disk full — the cache is ~34 GiB | Free space (the NAS helps), then re-run; verified files are skipped |
| Build-tool status `INCOMPLETE` | The 12 known gaps (gcc-12 cross compilers, stale constraints) | Ignore unless you rebuild those five sources; runtime is COMPLETE |

## Decoder ring

- **graybeard profile** — the graybeard preset from Tutorial 1: sources and build tools for the whole closure, unlimited depth
- **ensure-base** — ingest the installer DVD's index and register every package on it as the base set (fresh cache only)
- **provides** — a virtual name one package advertises as satisfied (e.g. `pkgconf` provides `pkg-config`); "no providers" means nothing in the index offers that name

## Next

[Tutorial 6 — Building Hyprland from source](6_hyprland_from_source.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*