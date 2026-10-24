# Tutorial 3 — Hyprland from backports, and the version-drift trap

◂ [Tutorial 2 — A newer kernel from backports](2_kernel_from_backports.md) · [All tutorials](README.md) · [Tutorial 4 — An offline NAS from a USB stick](4_simplest_ewaste_nas.md) ▸

Hyprland, the Wayland compositor, lives in backports — and backports versions drag their own shared libraries along. When those land on a box that was built from main, something has to give. This chapter installs Hyprland on the offline box, watches the version-drift trap remove your terminal emulator, and rebuilds one small library on the operator's side to get it back. Both stay apt-managed. Both stay offline.

> **Watch out** — backports packages that have a lot of shared dependencies can quickly amount into a broken system. We'll cover a better (though more involved) way to bring in such packages in later tutorials.

## At a glance

| | |
|---|---|
| **You'll end up with** | Hyprland installed from backports, kitty back and working — a locally rebuilt `libxkbcommon-x11-0` bridging the two |
| **You need** | Tutorials 1–2 done (cache + handoff live, kernel 7.1.8) · USB stick · admin + operator access · ~1.5 h patience for downloads |
| **Time** | ~30 min active, ~1.5 h waiting (downloads), ~10 min build on the operator box |
| **Difficulty** | ★★★ |
| **New concepts** | version drift, source package, build profile (nocheck), pin |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | adds Hyprland + the fix's source, ships diffs |
| **Operator** | ex-box (offline) | `op@ex-box ~$` | installs Hyprland, rebuilds one library, gets kitty back |

> In this tutorial the stick mounts at `/media/usb`; paths on the stick are written accordingly.

## Before you start

- [ ] Tutorials 1–2 complete — `ex-box` moniker exists, handoff live, diff cleared
- [ ] `project.local.toml` on bis declares `trixie-backports` (it does if you followed Tutorial 2)
- [ ] USB stick at hand
- [ ] You can become root on both machines

One heads-up before you dive in: **counts and totals drift between the recorded runs** as the cache grows (a "prior 211" here, "prior 212" in Tutorial 2). Trust the delta, not the running total.

## Milestone 1 — Add Hyprland, binaries and sources

Hyprland is only in backports, and installing it is going to pull a lot — including, later, buildable sources. So before adding it, tell `package_mgmt` to fetch sources next to binaries.

### Step 1.1 — (Optional) Pull a few main-repo extras first

If you want the usual Wayland sidekicks — a clipboard, a notification daemon, a wallpaper tool, screenshot tools — grab them from the *main* repo, not backports. Create the list:

```txt
# libglib2.0-bin and gsettings for theming
libglib2.0-bin
gsettings-desktop-schemas

# other packages I use
wl-clipboard
dunst
hyprpaper
hyprland-qtutils
grim
slurp
jq
```

```bash
vit@bis OpenFromSource$ pm add --file /tmp/hyprland_main.txt
```

You should get a normal summary — the exact `Packages requested`/`Resolved` counts vary with your cache. Then verify and download as usual. None of this is required for the trap story, so skip it if you'd rather go straight to Hyprland.

> **Good to know** — the count shown here says 6 packages resolved even though the list has more lines. Some entries resolve to already-cached packages or get merged; the resolved count reflects what's genuinely new. "Counts vary with your cache."

### Step 1.2 — Switch on the developer profile

Set `source = true` and a finite source depth so the cache stays small while the requested set and its immediate neighborhood remain rebuildable. Your `project.local.toml` gains two sections:

```toml
[fetch]
# developer: fetch sources (and their build tools) for the in-depth set.
source = true

[resolution]
# Depth convention: -1 = unlimited, 0 = none, positive N = up to N levels.
# 1 = the requested packages (level 0) and their direct runtime dependencies (level 1).
# A finite depth keeps the cache small while the requested set and its shallow neighborhood remain rebuildable.
max_source_dep_depth = 1
```

> **Say what? — "developer profile":** the developer preset from Tutorial 1 — `source = true` with a finite depth: `package_mgmt` additionally fetches each binary's *source* plus the build tools that source needs, so anything in the cache can be rebuilt — here, on the operator box.

### Step 1.3 — Add Hyprland from backports

```txt
hyprland
hyprland-guiutils
```

```bash
vit@bis OpenFromSource$ pm add --file /tmp/hyprland_backports.txt --source trixie-backports
```

<details><summary>Full add log</summary>

```console
vit@bis OpenFromSource$ pm add --file /tmp/hyprland_backports.txt --source trixie-backports
INFO:package_mgmt.lib.service.db_init:Database schema validated at /var/cache/package_mgmt/package_mgmt.db (v1)
INFO:package_mgmt.lib.service.resolver:Indexes loaded: 73148 binary packages, 38977 source packages (2 sources configured)
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libgcc-s1 -> libc6
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: openssl-provider-legacy -> libssl3t64
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libheif-plugin-dav1d -> libheif1
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libheif-plugin-libde265 -> libheif1
Resolved 2/2WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: python3-jaraco.context -> python3-setuptools
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: librust-memchr-2+std-dev -> librust-log-0.4+default-dev
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: librust-memchr-2-dev -> librust-log-0.4+default-dev
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: rake -> ruby
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: ruby-rubygems -> ruby
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: liblwp-protocol-https-perl -> libwww-perl

Packages requested:  2
Packages resolved:   2
Errors:              0
```

> **It's fine** — the `Circular dependency` warnings are the same Debian graph loops as Tutorial 1, joined this time by lines like `Resolved 2/2WARNING:…` where the progress marker and a warning collided on one terminal line. `Errors: 0` is the number that counts.

</details>

### Step 1.4 — Verify and download

```bash
vit@bis OpenFromSource$ pm list --deps hyprland
$ pm verify
```

Both should report `COMPLETE`. Then download — 993 binaries *and* 154 sources this time, because the developer profile is on:

```bash
vit@bis OpenFromSource$ pm download
```

```console
vit@bis OpenFromSource$ pm download
Elapsed: 00:00:32
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 21/993 files (8.1 MiB/946.8 MiB)
…
Elapsed: 00:53:35
[████████████████████████████████████████] 993/993 files (946.8 MiB/946.8 MiB)

Binary phase summary
────────────────────
Files downloaded:  993
Bytes:             946.8 MiB
Files verified:    993
Failed:            0
Elapsed:           00:53:35
…
Elapsed: 00:20:43
[████████████████████████████████████████] 154/154 files (341.7 MiB/341.7 MiB)

Source phase summary
────────────────────
Files downloaded:  154
Bytes:             341.7 MiB
Files verified:    154
Failed:            0
Elapsed:           00:20:43
…
Total: 1147 files downloaded, 1147 verified, 0 failed (01:14:18)
```

## Milestone 2 — Ship, install, and step in the trap

### Step 2.1 — Sync the diff

```bash
vit@bis OpenFromSource$ pm sync --system ex-box
```
Output

```console
vit@bis OpenFromSource$ pm sync --system ex-box
INFO:package_mgmt.lib.service.handoff:diff generated: 993 new across 2 release(s) (trixie, trixie-backports), 211 delivered in the base set
Sync ready:      /var/cache/package_mgmt/binary/diff
Release:         trixie, trixie-backports
Components:      trixie: main, contrib, non-free; trixie-backports: main
New packages:    993 (trixie/main: 930, trixie-backports/main: 63)
Total offered:   1204 (prior 211 + new 993)
Signed:          yes (key offline-cache.gpg, same as handoff)
Apply:           /var/cache/package_mgmt/binary/diff/apply.sh
Snapshot:        993 binaries marked for 'ex-box'
```

The sync clears any stale `diff` directory before writing the new one, so no housekeeping needed here. Copy `diff` onto the stick.

### Step 2.2 — Apply on the operator box

```bash
op@ex-box cache$ sudo cp -r diff /var/cache/package_mgmt/binary/
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
```
Output

```console
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
Apply: merging diff into /var/cache/package_mgmt/binary (releases: trixie, trixie-backports)
New files:       993
Offered after:   1204 packages (993 new)
Handoff indexes: replaced for trixie trixie-backports
apt update: OK
```

### Step 2.3 — Install Hyprland, watch kitty leave

```bash
op@ex-box ~$ sudo apt install hyprland hyprland-guiutils
```

Read the plan carefully before answering:

```console
op@ex-box ~$ sudo apt install hyprland hyprland-guiutils
The following packages were automatically installed and are no longer required:
  kitty-shell-integration  kitty-terminfo  libpython3.13  libxcb-xkb1
Use 'sudo apt autoremove' to remove them.

Upgrading:
  libxkbcommon0

Installing:
  hyprland  hyprland-guiutils

Installing dependencies:
  binutils  …  libxcb-errors0  …
…
REMOVING:
  kitty  libxkbcommon-x11-0

Summary:
  Upgrading: 1, Installing: 92, Removing: 2, Not Upgrading: 265
  Download size: 0 B / 64.8 MB
  Freed space: 23.9 MB

Continue? [Y/n]
```

**Hyprland installs, but `kitty` and `libxkbcommon-x11-0` are removed.** That's the version-drift trap:

> **Say what? — "version drift":** main and backports supply *different versions* of the same shared library. Hyprland from backports upgrades `libxkbcommon0` to 1.13.1; kitty from main depends on the *old* X11 helper, which itself pins the old core. The two can't agree, so apt removes kitty rather than break Hyprland.

> **Say what? — "libxkbcommon-x11-0":** the X11 bridge for the XKB keyboard library. kitty needs it; Hyprland's backports `libxkbcommon0` 1.13.1 makes the old main build unsatisfiable. The core library (`libxkbcommon0`) and its X11 helper are two separate binary packages built from one source.

For now, let it install — confirm `Y`. Hyprland is the point of this chapter. The details show the full install.

<details><summary>Full Hyprland install log</summary>

```console
op@ex-box ~$ sudo apt install hyprland hyprland-guiutils
The following packages were automatically installed and are no longer required:
  kitty-shell-integration  kitty-terminfo  libpython3.13  libxcb-xkb1
Use 'sudo apt autoremove' to remove them.

Upgrading:
  libxkbcommon0

Installing:
  hyprland  hyprland-guiutils

Installing dependencies:
  binutils           libduktape207            libheif1          liblerc4               libsensors-config    libxcb-errors0
  fontconfig         libegl-mesa0             libhwy1t64        libllvm19              libsensors5          libxcb-icccm4
  libabsl20240722    libegl1                  libhyprcursor0    liblua5.4-0            libsharpyuv0         libxcb-present0
  libaquamarine10    libevdev2                libhyprgraphics4  libmtdev1t64           libthai-data         libxcb-randr0
  libaquamarine9     libfribidi0              libhyprlang2      libmuparser2v5         libthai0             libxcb-res0
  libcairo-gobject2  libgbm1                  libhyprtoolkit5   libopengl0             libtiff6             libxcb-sync1
  libdatrie1         libgdk-pixbuf-2.0-0      libhyprutils10    libpango-1.0-0         libtomlplusplus3t64  libxcb-xfixes0
  libdav1d7          libgdk-pixbuf2.0-common  libhyprutils12    libpangocairo-1.0-0    libudis86-0          libxshmfence1
  libde265-0         libgl1-mesa-dri          libhyprwire3      libpangoft2-1.0-0      libvulkan1           libz3-4
  libdeflate0        libgles2                 libiniparser4     libpciaccess0          libwacom-common      libzip5
  libdisplay-info2   libglvnd0                libinput-bin      libpolkit-agent-1-0    libwacom9            mesa-libgallium
  libdrm-amdgpu1     libgprofng0              libinput10        libpolkit-gobject-1-0  libwayland-server0   polkitd
  libdrm-common      libgudev-1.0-0           libjbig0          libre2-11              libwebp7             sgml-base
  libdrm-intel1      libheif-plugin-dav1d     libjpeg62-turbo   librsvg2-2             libxcb-composite0    shared-mime-info
  libdrm2            libheif-plugin-libde265  libjxl0.11        libseat1               libxcb-dri3-0        xml-core

REMOVING:
  kitty  libxkbcommon-x11-0

Summary:
  Upgrading: 1, Installing: 92, Removing: 2, Not Upgrading: 265
  Download size: 0 B / 64.8 MB
  Freed space: 23.9 MB

Continue? [Y/n]
```

</details>

### Step 2.4 — Confirm the damage

Kitty isn't just uninstalled — apt can't even install it back:

```bash
op@ex-box ~$ sudo apt install kitty
```
Output

```console
op@ex-box ~$ sudo apt install kitty
Solving dependencies... Error!  
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

Unsatisfied dependencies:
 libxkbcommon-x11-0 : Depends: libxkbcommon0 (= 1.7.0-2) but 1.13.1-1~bpo13+1 is to be installed
Error: Unable to correct problems, you have held broken packages.
Error: The following information from --solver 3.0 may provide additional context:
   Unable to satisfy dependencies. Reached two conflicting decisions:
   1. libxkbcommon-x11-0:amd64 is selected for install because:
       1. kitty:amd64=0.41.1-2+deb13u2 is selected for install
       2. kitty:amd64 Depends libxkbcommon-x11-0 (>= 0.5.0)
   2. libxkbcommon-x11-0:amd64 Depends libxkbcommon0 (= 1.7.0-2)
      but none of the choices are installable:
      [no choices]
```

The core of the mess is the exact pin: the stock `libxkbcommon-x11-0` requires `libxkbcommon0 (= 1.7.0-2)` — *exactly* the old version — but the box has the backports `1.13.1`. The fix: build a `libxkbcommon-x11-0` that accepts any `libxkbcommon0 >= 1.7.0`. The box already has everything to do that — because the developer profile fetched the sources.

> **Good to know** — for a `lib` deep dive, feel free to check out the version drift trap article from the `docs`.

## Milestone 3 — Rebuild the helper on the operator box

### Step 3.1 — Admin: add the source (it comes along with its binary)

The package you need is the *binary* `libxkbcommon-x11-0`. Its **source** is named `libxkbcommon` — Debian builds `libxkbcommon0`, `libxkbcommon-x11-0`, `libxkbcommon-dev` and friends all from that one source tree. There is no source named `libxkbcommon-x11-0`, so ask for the binary, from **main** (not backports — you're rebuilding the main version):

```bash
vit@bis OpenFromSource$ pm add libxkbcommon-x11-0
```

> **Say what? — "source package":** the trio of files (`libxkbcommon_1.7.0-2.dsc`, `.orig.tar.xz`, `.debian.tar.xz`) that `dpkg` needs to build the binaries. One source builds several binary packages; here you're rebuilding just one of them.

With `source = true`, this links the binary to the `libxkbcommon` source and resolves that source's `Build-Depends` into installable binaries. Look at what the operator will need to build it — this tree is the authoritative build-tool list:

```bash
vit@bis OpenFromSource$ pm list --deps libxkbcommon-x11-0 --include-build-deps
```

<details><summary>Full dependency tree including build deps</summary>

```console
libxkbcommon-x11-0 (current)
  ├─ libxkbcommon0 (= 1.7.0-2) [binary ✓, source ✓, verified: ✗]
  │ ├─ xkb-data (= 2.42-1) [binary ✓, source: missing, verified: ✓]
  │ └─ libc6 (= 2.41-12+deb13u4) [binary ✓, source ✓, verified: ✓]
  │   └─ libgcc-s1 (= 14.2.0-19) [binary ✓, source ✓, verified: ✓]
  │     ├─ gcc-14-base (= 14.2.0-19) [binary ✓, source: missing, verified: ✓]
  ├─ libxcb-xkb1 (= 1.17.0-2+b1) [binary ✓, source ✓, verified: ✓]
  │ └─ libxcb1 (= 1.17.0-2+b1) [binary ✓, source ✓, verified: ✓]
  │   ├─ libxau6 (= 1:1.0.11-1) [binary ✓, source: missing, verified: ✓]
  │   └─ libxdmcp6 (= 1:1.1.5-1) [binary ✓, source: missing, verified: ✓]
  ├─ [build] xkb-data (= 2.42-1) [binary ✓, verified: ✓]
  ├─ [build] debhelper (= 13.24.2) [binary ✓, verified: ✗]
  ├─ [build] pkgconf (= 1.8.1-4) [binary ✓, verified: ✓]
  ├─ [build] meson (= 1.7.0-1) [binary ✓, verified: ✓]
  ├─ [build] quilt (= 0.68-1) [binary ✓, verified: ✓]
  ├─ [build] bison (= 2:3.8.2+dfsg-1+b2) [binary ✓, verified: ✓]
  ├─ [build] x11proto-dev (= 2024.1-1) [binary ✓, verified: ✓]
  ├─ [build] flex (= 2.6.4-8.2+b4) [binary ✓, verified: ✓]
  ├─ [build] libxml2-dev (= 2.12.7+dfsg+really2.9.14-2.1+deb13u3) [binary ✓, verified: ✓]
  ├─ [build] doxygen (= 1.9.8+ds-2.1) [binary ✓, verified: ✓]
  ├─ [build] graphviz (= 2.42.4-3) [binary ✓, verified: ✓]
  ├─ [build] x11-xkb-utils (= 7.7+9) [binary ✓, verified: ✓]
  ├─ [build] xvfb (= 2:21.1.16-1.3+deb13u3) [binary ✓, verified: ✓]
  ├─ [build] dh-exec (= 0.30) [binary ✓, verified: ✓]
  └─ [build] libxcb-xkb-dev (= 1.17.0-2+b1) [binary ✓, verified: ✓]
```

</details>

> **Watch out** — `pm add libxkbcommon` (the source name) would *not* work: there's no binary package by that name, so the resolver reports it as an unresolvable virtual package. The source rides along when you add one of its binaries.

### Step 3.2 — Admin: download, verify, ship

```bash
vit@bis OpenFromSource$ pm download
```

<details><summary>Full download log</summary>

```console
vit@bis OpenFromSource$ pm download
Elapsed: 00:00:58
[████████████████████████████████████████] 9/9 files (14.4 MiB/14.4 MiB)

Binary phase summary
────────────────────
Files downloaded:  9
Bytes:             14.4 MiB
Files verified:    9
Failed:            0
Elapsed:           00:00:58
…
Elapsed: 00:00:47
[████████████████████████████████████████] 6/6 files (11.9 MiB/11.9 MiB)

Source phase summary
────────────────────
Files downloaded:  6
Bytes:             11.9 MiB
Files verified:    6
Failed:            0
Elapsed:           00:00:47
…
Total: 15 files downloaded, 15 verified, 0 failed (00:01:45)
```
</details>

The six source files are the `libxkbcommon_1.7.0-2.dsc`, `.orig.tar.xz` and `.debian.tar.xz` (plus the build-tool sources). Next verify — and don't worry about the result:

```bash
vit@bis OpenFromSource$ pm verify
```

<details><summary>Full verification log</summary>

```console
Build-Tool Dependencies
───────────────────────
Total build-dep links:  718
Total build tools:      309
Unresolved build deps:  38
  - libelf-dev — usable record exists but not linked for matching version (required by: glib2.0)
  - python3-debian — usable record exists but not linked for matching version (required by: glib2.0)
  - wayland-protocols — usable record exists but not linked for matching version (required by: wl-clipboard)
  - cmake — usable record exists but not linked for matching version (required by: hyprpaper)
<SNIP — 34 more lines omitted>
Missing build tools:    0
Build Status:           INCOMPLETE

Status:                 INCOMPLETE
```

> **It's fine** — the `INCOMPLETE` build-tool status. It means 38 declared build-dependencies point at *superseded* versions — stale links left over from mixing main and backports in one cache, not missing tools (`Missing build tools: 0`, 309 tools present). The runtime graph is `COMPLETE`, and the main `libxkbcommon` source you're about to rebuild is fully satisfied. For the story, ignore it.

</details>

Now sync the fix and put the source files on the stick:

```bash
vit@bis OpenFromSource$ pm sync --system ex-box
```
Output

```console
vit@bis OpenFromSource$ pm sync --system ex-box
INFO:package_mgmt.lib.service.handoff:diff generated: 19 new across 2 release(s) (trixie, trixie-backports), 1195 delivered in the base set
Sync ready:      /var/cache/package_mgmt/binary/diff
Release:         trixie, trixie-backports
Components:      trixie: main, contrib, non-free; trixie-backports: main
New packages:    19 (trixie/main: 19)
Total offered:   1214 (prior 1195 + new 19)
Signed:          yes (key offline-cache.gpg, same as handoff)
Apply:           /var/cache/package_mgmt/binary/diff/apply.sh
Snapshot:        19 binaries marked for 'ex-box'
```

> **Good to know** — the diff carries **binaries only**; sources live in their own pool and don't travel in `diff/`. Copy the three `libxkbcommon_*` source files onto the stick separately, keeping the pool layout — say under `sources/main/lib/libxkbcommon/`.

### Step 3.3 — Operator: apply the diff, install the build tools

```bash
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
$ sudo apt update
```

Now install the build tools. First attempt — this one will fail, and that's the lesson:

```bash
op@ex-box ~$ sudo apt install bison debhelper dh-exec doxygen flex graphviz \
    libxcb-xkb-dev libxml2-dev meson pkgconf quilt \
    x11-xkb-utils x11proto-dev xkb-data xvfb
```

<details><summary>Full build-deps failure log</summary>

```console
op@ex-box ~$ sudo apt install bison debhelper dh-exec doxygen flex graphviz \
     libxcb-xkb-dev libxml2-dev meson pkgconf quilt \
     x11-xkb-utils x11proto-dev xkb-data xvfb
Solving dependencies... Error!  
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

Unsatisfied dependencies:
 libglx-mesa0 : Depends: mesa-libgallium (= 25.0.7-2+deb13u1) but 26.1.2-1~bpo13+1 is to be installed
Error: Unable to correct problems, you have held broken packages.
Error: The following information from --solver 3.0 may provide additional context:
   Unable to satisfy dependencies. Reached two conflicting decisions:
   1. mesa-libgallium:amd64=25.0.7-2+deb13u1 is not selected for install
   2. mesa-libgallium:amd64=25.0.7-2+deb13u1 is selected as a downgrade because:
       1. xvfb:amd64=2:21.1.16-1.3+deb13u3 is selected for install
       2. xvfb:amd64 Depends libgl1
       3. libgl1:amd64 Depends libglx0 (= 1.7.0-1+b2)
       4. libglx0:amd64 Depends libglx-mesa0
       5. libglx-mesa0:amd64 Depends mesa-libgallium (= 25.0.7-2+deb13u1)
```

</details>

**The same trap, one level down.** `xvfb` (from main) drags in the whole GL stack, which would force Hyprland's backports `mesa-libgallium` 26.1.2 back down to the main 25.0.7. apt refuses to break Hyprland. But `xvfb` — like `x11-xkb-utils` and `xkb-data` — is marked `<!nocheck>` in `debian/control`: it exists only to run the test suite. Skip the tests and you don't need any of them. Install the real build tools without `xvfb`:

```bash
op@ex-box ~$ sudo apt install build-essential bison debhelper dh-exec doxygen flex graphviz \
    libwayland-dev libxcb-xkb-dev libxml2-dev meson pkgconf quilt \
    wayland-protocols x11proto-dev
```

> **Good to know** — two names you might expect are served indirectly: `pkg-config` is provided by `pkgconf`, and `ninja-build` is pulled in automatically by `meson`. Also, there's no `deb-src` line in the cache repo, so `apt build-dep` can't run — but you don't need it: the developer resolution already turned every build dependency into an ordinary binary you install by name.

### Step 3.4 — Operator: unpack the source

```bash
op@ex-box ~$ mkdir -p ~/src && cd ~/src
op@ex-box ~/src$ cp /media/usb/sources/main/lib/libxkbcommon/* .
op@ex-box ~/src$ dpkg-source -x libxkbcommon_1.7.0-2.dsc
op@ex-box ~/src$ cd libxkbcommon-1.7.0
```

### Step 3.5 — Operator: relax the pin

The `libxkbcommon-x11-0` stanza in `debian/control` pins the core library to the exact built version. That's the pin that broke kitty. Change it to accept anything from the same generation:

```diff
 Depends:
- libxkbcommon0 (= ${binary:Version}),
+ libxkbcommon0 (>= 1.7.0),
  ${shlibs:Depends},
  ${misc:Depends}
```

> **Say what? — "pin":** a version constraint like `(= 1.7.0-2)` that forces a package to be used with exactly one version of another. The whole trap is two packages pinning each other across suites. `>= 1.7.0` is satisfied by the installed backports `1.13.1` while keeping the helper and core on the same API generation.

### Step 3.6 — Operator: build, skipping the tests

```bash
op@ex-box ~/src/libxkbcommon-1.7.0$ DEB_BUILD_OPTIONS=nocheck DEB_BUILD_PROFILES=nocheck \
    dpkg-buildpackage -us -uc -b
op@ex-box ~/src/libxkbcommon-1.7.0$ cd ..
```

> **Say what? — "build profile" (nocheck):** `DEB_BUILD_OPTIONS=nocheck DEB_BUILD_PROFILES=nocheck` tells the package's build system to skip the test suite. That's what `xvfb` and the other `<!nocheck>` build-deps were for — no tests, no GL stack, no downgrade. `-us -uc -b` skips signing and builds only the binary packages.

### Step 3.7 — Operator: install only the X11 helper

```bash
op@ex-box ~/src$ sudo apt install ./libxkbcommon-x11-0_1.7.0-2_amd64.deb
```

> **Watch out** — **Don't install** `libxkbcommon0_1.7.0-2_amd64.deb`. Installing it would downgrade the core library and break Hyprland. Only the X11 helper gets the relaxed pin.

<details><summary>Full install log</summary>

```console
op@ex-box src$ sudo apt install ./libxkbcommon-x11-0_1.7.0-2_amd64.deb 
Note, selecting 'libxkbcommon-x11-0' instead of './libxkbcommon-x11-0_1.7.0-2_amd64.deb'
Installing:
  libxkbcommon-x11-0

Summary:
  Upgrading: 0, Installing: 1, Removing: 0, Not Upgrading: 469
  Download size: 0 B / 16.1 kB
  Space needed: 58.4 kB / 467 GB available

Get:1 /home/op/src/libxkbcommon-x11-0_1.7.0-2_amd64.deb libxkbcommon-x11-0 amd64 1.7.0-2 [16.1 kB]
Selecting previously unselected package libxkbcommon-x11-0:amd64.
(Reading database ... 70059 files and directories currently installed.)
Preparing to unpack .../libxkbcommon-x11-0_1.7.0-2_amd64.deb ...
Unpacking libxkbcommon-x11-0:amd64 (1.7.0-2) ...
Setting up libxkbcommon-x11-0:amd64 (1.7.0-2) ...
Processing triggers for libc-bin (2.41-12+deb13u4) ...
Notice: Download is performed unsandboxed as root as file '/home/op/src/libxkbcommon-x11-0_1.7.0-2_amd64.deb' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)
```

> **It's fine** — the `Notice: Download is performed unsandboxed as root` line. apt is grumbling that the `_apt` user can't read a file in `/home/op`; the install itself succeeded.

</details>

### Step 3.8 — Operator: bring kitty back

```bash
op@ex-box ~/src$ sudo apt install kitty
```

```console
op@ex-box ~/src$ sudo apt install kitty
Installing:
  kitty

Installing dependencies:
  kitty-shell-integration  kitty-terminfo  libpython3.13

Summary:
  Upgrading: 0, Installing: 4, Removing: 0, Not Upgrading: 470
  Download size: 0 B / 9,613 kB
  Space needed: 0 B / 467 GB available

Continue? [Y/n]
```

kitty's `libxkbcommon-x11-0 (>= 0.5.0)` is satisfied by your rebuilt package, which in turn accepts the backports `libxkbcommon0`. Kitty is back, Hyprland stays, and both remain apt-managed.

## Checkpoint — prove both live together

```console
op@ex-box ~$ kitty --version
kitty 0.41.1 created by Kovid Goyal
op@ex-box ~$ dpkg -l | grep -E 'kitty|libxkbcommon'
ii  kitty                            0.41.1-2+deb13u2                     amd64        fast, featureful, GPU based terminal emulator
ii  kitty-shell-integration          0.41.1-2+deb13u2                     all          fast, featureful, GPU based terminal emulator (shell integration files)
ii  kitty-terminfo                   0.41.1-2+deb13u2                     all          fast, featureful, GPU based terminal emulator (terminfo file)
ii  libxkbcommon-x11-0:amd64         1.7.0-2                              amd64        library to create keymaps with the XKB X11 protocol
ii  libxkbcommon0:amd64              1.13.1-1~bpo13+1                     amd64        library interface to the XKB compiler - shared library
```

The two packages you could never install together in Milestone 2 are now both installed: `libxkbcommon-x11-0` at the locally rebuilt `1.7.0-2`, `libxkbcommon0` still on backports `1.13.1`. `apt-cache policy` confirms the split is stable — details below. Log into the Hyprland session and open kitty under Wayland to confirm end to end.

<details><summary>apt-cache policy output</summary>

```console
op@ex-box ~$ apt-cache policy libxkbcommon0 libxkbcommon-x11-0
libxkbcommon0:
  Installed: 1.13.1-1~bpo13+1
  Candidate: 1.13.1-1~bpo13+1
  Version table:
 *** 1.13.1-1~bpo13+1 100
        100 /var/lib/dpkg/status
     1.7.0-2 500
        500 file:/var/cache/package_mgmt/binary/handoff trixie/main amd64 Packages
libxkbcommon-x11-0:
  Installed: 1.7.0-2
  Candidate: 1.7.0-2
  Version table:
     1.7.0-2 500
        500 file:/var/cache/package_mgmt/binary/handoff trixie/main amd64 Packages
 *** 1.7.0-2 100
        100 /var/lib/dpkg/status
```

</details>

> **Good to know** — until backports ships a `libxkbcommon-x11-0` that matches its own `libxkbcommon0`, a future `apt upgrade` might try to replace your local build. If it does, `apt-mark hold libxkbcommon-x11-0` stops it — and Tutorial 7 shows the proper endgame: rebuild the whole stack to align it with stock Debian (no backports) and import it into the cache, so the operator gets it through the normal handoff.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `apt install hyprland` removes kitty | Version drift, exactly as shown | Follow Milestone 3 — rebuild `libxkbcommon-x11-0` with the relaxed pin |
| `apt install kitty` says "held broken packages" | The exact `(= 1.7.0-2)` pin vs backports `1.13.1` | Same — that's the trap working as designed |
| Build-deps install fails on `mesa-libgallium` | `xvfb` in the list drags the GL stack down | Drop `xvfb` and build with the `nocheck` profile |
| `dpkg-source -x` complains about a missing file | Source files weren't copied to the stick | Copy all three `libxkbcommon_*` files, keeping the layout |
| Built `.deb` still pins `(= ${binary:Version})` | The pin is generated at build time, not in `control` | Check `dpkg -I` on the `.deb`; relax with `dpkg-deb -R` if needed |
| Installing `libxkbcommon0_1.7.0-2` breaks Hyprland | You installed the core, not just the helper | Reinstall the backports `libxkbcommon0` from the cache |

## Decoder ring

- **developer profile** — `source = true` (plus a source depth): fetch each binary's source and build tools so anything can be rebuilt
- **version drift** — main and backports provide different versions of the same shared library; a package pinned to one can break under the other
- **source package** — the `.dsc` + tarballs `dpkg` needs to build the binaries; one source builds several binary packages
- **build profile (nocheck)** — `DEB_BUILD_OPTIONS=nocheck DEB_BUILD_PROFILES=nocheck`: skip the test suite (and its `<!nocheck>` dependencies)
- **pin** — a version constraint like `(= 1.7.0-2)` forcing one package to match exactly another

## Next

[Tutorial 4 — An offline NAS from a USB stick](4_simplest_ewaste_nas.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*