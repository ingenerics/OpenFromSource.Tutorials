# Tutorial 6 — Building Hyprland from source

◂ [Tutorial 5 — Ensuring the full base set from the installer DVD](5_ensure_base_packages.md) · [All tutorials](../README.md) · [Tutorial 7 — Importing Hyprland into the cache](7_import_hyprland_stack.md) ▸

The backports Hyprland in Tutorial 3 was a prebuilt black box: you couldn't rebuild it, and it dragged a version-drift mess behind it. This is the alpha endgame — from the full base cache, pin the exact Hyprland era, clone the twelve source repos, fix the two portal build bugs, and compile the whole stack bottom-up with nothing but the cache. By the end: **build complete: the Hyprland stack is installed at /usr**.

> **Good to know** — from here on out the tutorials are very Hyprland focussed. Mainly because this is the use case I had in mind from the get-go. On top of that, the actual code for the commands we're about to use (`align-repo` and `package-stack`) are not (as) generic (as they could be). This was deliberate, however, as it squashes complexity. With present day tooling we should be able to transform these commands towards other use cases. That's how they should be approached. Think harness skills for scaffolding, instead of overengineering at early alpha stage.

## At a glance

| | |
|---|---|
| **You'll end up with** | The complete Hyprland stack compiled from pinned sources and installed at `/usr`, built entirely from the base cache |
| **You need** | Tutorial 5 done (graybeard base cache) · ~10 GB free for repos + build dirs · internet for the one-time clone |
| **Time** | ~30 min active, ~40 min waiting (build) |
| **Difficulty** | ★★★ |
| **New concepts** | era, manifest, align-repo, submodule |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | pins the era, acquires the tree, builds the stack |

## Before you start

- [ ] Tutorial 5 complete — graybeard base cache initialized and downloaded
- [ ] ~10 GB free disk on bis
- [ ] You can become root (the build installs to `/usr`)

## Milestone 1 — Pick the era and pin the tree

Hyprland releases are *eras*: each version (0.49, 0.50, …) moves the whole stack together — Hyprland, its companion libraries, the portal — as one locked set. The choice of era matters more than any package version you've pinned so far, because the newest eras won't compile on the frozen trixie tree.

### Step 1.1 — Understand why era 0.49

The alignment gate picks the newest era whose *declared* requirements fit the frozen tree, which would be era 0.50. But the build is the final arbiter: eras 0.50+ fail at compile time on requirements the gate can't see, and the era is therefore **pinned at 0.49** in the recorded manifest.

> **Good to know** — I left the behavior described above intact, because it can pay to stretch it just a little. If you only need to bring in a couple of build dependencies (manually), it might still be something to consider.

<details><summary>Why 0.49 and not 0.50+</summary>

**Toolchain and library drift (build-time).** The gate reads declared dependencies, so it cannot see requirements that live in the source itself. Building era 0.52 exposed several such requirements, all of which the frozen tree cannot satisfy without drift:

| Requirement (eras 0.50–0.52) | Frozen tree | Where it surfaces | Verdict |
|---|---|---|---|
| gcc-15 `#embed` (C++26) | gcc-14 14.2.0 lacks it | `src/config/defaultConfig.hpp` (0.50+) | toolchain drift |
| gcc-15 `std::string + std::string_view` (P2591) | gcc-14 14.2.0 lacks it | `hyprctl/main.cpp` (0.50+) | toolchain drift |
| `libxkbcommon>=0.8` API | `libxkbcommon-dev 1.7.0-2` | `src/devices/IKeyboard.cpp` (`XKB_KEYMAP_FORMAT_TEXT_V2`, `xkb_keymap_new_from_names2`) — 0.52 only | library drift |
| `?:` operands `CXCBConnection` vs `nullptr_t` | genuine type error, 0.50+ | `src/xwayland/XWM.hpp:217` | upstream bug |

Hyprland v0.50.0 (2025-07-16) migrated the stack to C++26 features that need GCC 15 (not in the frozen tree); v0.52.0 additionally calls libxkbcommon 0.8 APIs. Era **0.49** (2025-05-08) predates all of these — it targets `-std=c++26` but uses only the features gcc-14 implements, and needs only the xkbcommon 1.7 API — so it is the newest era that builds on the frozen gcc-14 + xkbcommon-1.7 tree.

</details>

### Step 1.2 — Clone the twelve repos into `repos/`

From the project root:

```bash
vit@bis OpenFromSource$ mkdir -p repos
$ for r in Hyprland hyprwayland-scanner hyprlang hyprutils hyprcursor \
         hyprgraphics aquamarine hyprland-protocols hyprpaper \
         hypridle hyprlock xdg-desktop-portal-hyprland; do
  git clone --quiet "https://github.com/hyprwm/$r" "repos/$r"
done
```

> **Say what? — "manifest":** the git-tracked version manifest, `manifests/hyprland-0.49.toml` — the exact commit of every repo at the aligned era, plus the alignment's rationale. Combined with this procedure it's the reproducible recipe for the pinned tree.

### Step 1.3 — Watch the gate refuse the fresh clones

Right after cloning, the repos sit at `HEAD`, not at the pinned commits. `align-repo --verify` checks every repo against the manifest and the cache — and refuses:

```bash
vit@bis OpenFromSource$ pm align-repo --verify manifests/hyprland-0.49.toml --repo-dir repos/
```
Output

```console
vit@bis OpenFromSource$ pm align-repo --verify manifests/hyprland-0.49.toml --repo-dir repos/
{"error": "repo Hyprland HEAD (156276e5144d) does not match manifest commit (9958d297641b)", "exit_code": 102}
```

> **It's fine** — this is the gate working. It verifies the repos are at the right commits for the frozen trixie tree; freshly cloned repos are at `HEAD`, so it names the mismatch and refuses to validate a partial tree.

> **Say what? — "align-repo":** the command that determines or verifies the version-aligned source set — whether an era's requirements are satisfied by the frozen tree plus the acquired sources.

### Step 1.4 — Pin each repo to its manifest commit

```bash
$ manifest=$(ls manifests/hyprland-*.toml)
$ for d in repos/*/; do
  name=$(basename "$d")
  commit=$(python3 -c "import sys,tomllib;[print(r['commit']) for r in tomllib.load(open(sys.argv[1],'rb'))['repo'] if r['name']=='$name']" "$manifest")
  git -C "$d" checkout --detach "$commit"
done
```

You should get a detach line per repo:

```console
HEAD is now at 1c8fa0b version: bump to 0.8.0
HEAD is now at 028bedb version: bump to 0.1.12
HEAD is now at 9d7f268 version: bump to 0.1.3
HEAD is now at 84f9f2e version: bump to 0.1.6
HEAD is now at 9958d297 version: bump to 0.49.0
HEAD is now at 3a5c2bd version: bump to 0.6.4
HEAD is now at 5572417 version: bump to 0.6.3
HEAD is now at 6c64630 version: bump to 0.8.2
HEAD is now at 753ffa7 version: bump to 0.7.5
HEAD is now at 674ea57 version: bump to 0.7.1
HEAD is now at de91347 version: bump to 0.4.4
HEAD is now at fa50ab3 version: bump to 1.3.9
```

### Step 1.5 — Capture submodules at their recorded commits

Some pinned refs vendor submodules. The important one: `xdg-desktop-portal-hyprland` v1.3.9 vendors **`sdbus-c++`** as `subprojects/sdbus-cpp` — and `sdbus-c++` is not packaged in trixie, so it's built from source as the stack's one recorded dependency, riding the cache via this submodule.

> **Say what? — "submodule":** a nested git repo pinned to a specific commit inside another repo — the way vendored sources ride along.

```bash
$ for d in repos/*/; do git -C "$d" submodule update --init --recursive; done
```

<details><summary>Full submodule log</summary>

```console
vit@bis OpenFromSource$ for d in repos/*/; do git -C "$d" submodule update --init --recursive; done
Submodule 'subprojects/hyprland-protocols' (https://github.com/hyprwm/hyprland-protocols) registered for path 'subprojects/hyprland-protocols'
Submodule 'subprojects/tracy' (https://github.com/wolfpld/tracy) registered for path 'subprojects/tracy'
Submodule 'subprojects/udis86' (https://github.com/canihavesomecoffee/udis86) registered for path 'subprojects/udis86'
Cloning into '/home/vit/code/OpenFromSource/repos/Hyprland/subprojects/hyprland-protocols'...
Cloning into '/home/vit/code/OpenFromSource/repos/Hyprland/subprojects/tracy'...
Cloning into '/home/vit/code/OpenFromSource/repos/Hyprland/subprojects/udis86'...
Submodule path 'subprojects/hyprland-protocols': checked out '3a5c2bda1c1a4e55cc1330c782547695a93f05b2'
Submodule path 'subprojects/tracy': checked out '37aff70dfa50cf6307b3fee6074d627dc2929143'
Submodule path 'subprojects/udis86': checked out '5336633af70f3917760a6d441ff02d93477b0c86'
Submodule 'hyprland-protocols' (https://github.com/hyprwm/hyprland-protocols) registered for path 'subprojects/hyprland-protocols'
Submodule 'subprojects/sdbus-cpp' (https://github.com/Kistler-Group/sdbus-cpp) registered for path 'subprojects/sdbus-cpp'
Cloning into '/home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/subprojects/hyprland-protocols'...
Cloning into '/home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/subprojects/sdbus-cpp'...
Submodule path 'subprojects/hyprland-protocols': checked out '4d29e48433270a2af06b8bc711ca1fe5109746cd'
Submodule path 'subprojects/sdbus-cpp': checked out 'b7d85f936d622299ee57966c2b2aefde5e4f3684'
```

</details>

### Step 1.6 — Verify the whole tree

You can manually check every `HEAD`, the submodule pins, the `sdbus-c++` tag, or just ask the gate again — the last command is the one that matters:

```bash
$ for d in repos/*/; do echo "$(basename "$d"): $(git -C "$d" rev-parse HEAD)"; done
$ git -C repos/Hyprland submodule status
$ git -C repos/xdg-desktop-portal-hyprland submodule status
$ git -C repos/xdg-desktop-portal-hyprland/subprojects/sdbus-cpp describe --tags --exact-match
$ pm align-repo --verify manifests/hyprland-0.49.toml --repo-dir repos/
```
Output

```console
vit@bis OpenFromSource$ pm align-repo --verify manifests/hyprland-0.49.toml --repo-dir repos/
Alignment (verify): manifests/hyprland-0.49.toml
  baseline: trixie/main (verified, read-only)
  requirements checked: 68
  result: PASS
```

**Expected**: every `HEAD` equals its manifest SHA; the `sdbus-cpp` submodule is at tag `v2.0.0`; `result: PASS`. From here on out, nothing in this chapter needs the network.

> **Watch out** — a mismatched tree is refused: `align-repo --verify` exits 102 and names the offending repo rather than validate a partial tree. `repos/` is git-ignored and reproducible — re-run Milestone 1 to restore it any time.

## Milestone 2 — Fix the two portal build bugs

The portal's **meson** build (as opposed to its CMake one) has two omissions that are fixed upstream only in v1.4.0: `XDPH_VERSION` is never defined, and the `hyprland-share-picker` target links Qt but not `hyprutils`. The pinned v1.3.9 tree needs both backported before it will build:

```bash
vit@bis OpenFromSource$ python3 - <<'EOF'
p = "repos/xdg-desktop-portal-hyprland/meson.build"
s = open(p).read()
if "XDPH_VERSION" not in s:
    s = s.replace(
        "conf_data = configuration_data()",
        'version = meson.project_version()\n\n'
        'add_project_arguments(f\'-DXDPH_VERSION="{version}"\', language : \'cpp\')\n\n'
        'conf_data = configuration_data()',
    )
    open(p, "w").write(s)
p = "repos/xdg-desktop-portal-hyprland/hyprland-share-picker/meson.build"
s = open(p).read()
if "dependency('hyprutils')" not in s:
    s = s.replace(
        "  dependencies: qtdep,",
        "  dependencies: [qtdep, dependency('hyprutils')],",
    )
    open(p, "w").write(s)
EOF
```

> **Good to know** — I didn't include this in `scripts/build-hyprland.sh` to make it stand out next time I revisit everything for the next Debian LTS.

## Milestone 3 — Build the stack bottom-up from the cache

Every toolchain package and development library comes from the cache at the frozen trixie versions — meson 1.7.0, ninja 1.12.1, cmake 3.31.6, gcc-14 14.2.0. Install them all in one go:

```bash
vit@bis OpenFromSource$ sudo apt install meson ninja-build cmake gcc g++ build-essential \
    libwayland-dev libdrm-dev libgbm-dev glslang-dev libinput-dev \
    libxkbcommon-dev libglib2.0-dev libpango1.0-dev libcairo2-dev \
    libpixman-1-dev libxcursor-dev uuid-dev libeis-dev liblcms2-dev \
    libcanberra-dev libre2-dev libmuparser-dev wayland-protocols xwayland \
    libseat-dev libdisplay-info-dev libtomlplusplus-dev libsystemd-dev \
    libspdlog-dev libfmt-dev nlohmann-json3-dev \
    libxcb1-dev libxcb-render0-dev libxcb-xfixes0-dev libxcb-icccm4-dev \
    libxcb-composite0-dev libxcb-res0-dev libxcb-errors-dev libudev-dev hwdata libzip-dev \
    librsvg2-dev libjpeg-dev libwebp-dev libmagic-dev libjxl-dev libspng-dev \
    libpugixml-dev libegl-dev libopengl-dev libgles-dev libpng-dev libpam0g-dev \
    qt6-base-dev libpipewire-0.3-dev libspa-0.2-dev
```

> **Good to know** — three names are *deliberately* absent, and one is extra: `libsdbus-c++-dev` and `libglaze-dev` aren't packaged in trixie (sdbus-c++ is built from source as the recorded dependency; Hyprland's `hyprpm` is disabled at configure time, so no glaze); `hyprland-protocols` is built from source as component 1 of the stack. `libxcb-errors-dev`, by contrast, is required even though meson marks it `required: auto` — without it, configure passes but the XWayland compile fails.

Now the whole stack, bottom-up in one command. Build order: protocols → scanner → companion libraries → Hyprland → the source-built `sdbus-c++` → companion tools. The build takes a while:

```bash
vit@bis OpenFromSource$ ./scripts/build-hyprland.sh
```

<details><summary>For extra points — a headless Hyprland</summary>

Want to use Hyprland as a headless desktop (virtualized setup, remoting in)? See https://github.com/hyprwm/Hyprland/pull/13616 (and its companion in aquamarine, https://github.com/hyprwm/aquamarine/pull/255). Spoiler alert; there's a little more to it than meets the eye, at least on trixie. Still wanted to plant a seed, even though it would lead us too far for this tutorial.

</details>

## Checkpoint — the stack is built

Your final output should end with the portal's install and the proof line:

```console
vit@bis OpenFromSource$ ./scripts/build-hyprland.sh
…
==> [xdg-desktop-portal-hyprland] install
ninja: Entering directory `repos/xdg-desktop-portal-hyprland/build'
[0/1] Installing files
Installing src/xdg-desktop-portal-hyprland to /usr/libexec
Installing hyprland-share-picker/hyprland-share-picker to /usr/bin
Installing /home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/build/org.freedesktop.impl.portal.desktop.hyprland.service to /usr/share/dbus-1/services
Installing /home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/hyprland.portal to /usr/share/xdg-desktop-portal/portals
build complete: the Hyprland stack is installed at /usr
```

The whole stack — Hyprland, hyprwayland-scanner, hyprlang, hyprutils, hyprcursor, hyprgraphics, aquamarine, hyprland-protocols, hyprpaper, hypridle, hyprlock, the source-built sdbus-c++, and the portal — compiled from pinned sources against the frozen tree. Next step is the one Tutorial 7 is about: instead of installing straight to `/usr`, stage the build and package it into the cache so the offline operators get it through plain `apt`.

> **Good to know** — the build script has a few handy controls: `--from Hyprland` resumes from a component, `--only hyprlang` builds just one, `--rebuild` wipes each component's build dir for a full rebuild. And `--stage DIR` (Tutorial 7) installs into a staging root instead of `/usr` — no root needed.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `align-repo --verify` exits 102 naming a repo | Repos at `HEAD`, not the pinned commit (or a re-acquisition gone wrong) | Re-run Milestone 1 steps 1.4–1.6 |
| Submodule clone fails partway | Partial acquisition | Re-run `submodule update --init --recursive` |
| `align-repo --verify` reports `FAIL` naming requirements | Tree or cache changed so even era 0.49 no longer aligns | Don't build — restore the pinned tree / cache, then re-verify |
| Portal compile fails on `XDPH_VERSION` / share-picker link | The two meson backports not applied | Re-run Milestone 2 (or rely on the script's auto-apply) |
| XWayland compile fails despite configure passing | `libxcb-errors-dev` missing (meson says `auto`, reality says required) | Install it from the cache and rebuild |
| Build fails on a missing build helper | A tool the stack needs that isn't in the cache | Add it to the cache (per the dependency-graph rule) and rebuild |

## Decoder ring

- **era** — a Hyprland release treated as a versioned set; the whole stack moves together
- **manifest** — the git-tracked version manifest recording every repo's exact commit at the aligned era
- **align-repo** — determine or verify the version-aligned source set against the frozen tree + acquired sources
- **submodule** — a nested git repo pinned to a commit inside another repo (vendored sources)

## Next

[Tutorial 7 — Importing Hyprland into the cache](7_import_hyprland_stack.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*
