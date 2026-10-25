# Tutorial 7 — Importing Hyprland into the cache

◂ [Tutorial 6 — Building Hyprland from source](6_hyprland_from_source.md) · [All tutorials](../README.md) ▸

Tutorial 6 compiled the Hyprland stack and installed it to `/usr` on the build box. But the end goal are the offline machines. This chapter stages the same build, packages all thirteen components into proper `.deb` + source pairs, and imports them into the cache's `local` component — so operators install Hyprland with plain `apt`, offline, no `dpkg -i`, and every shipped binary is linked to a rebuildable source. It also pays off the promise from Tutorial 3: your one-off rebuild can ride the cache like everything else.

## At a glance

| | |
|---|---|
| **You'll end up with** | The Hyprland stack as 13 verified `.deb` + `.dsc` pairs in the cache's `local` component; operators run `apt install hyprland …` and it just works |
| **You need** | Tutorial 6 done (repos/ pinned at era 0.49, toolchain installed) · graybeard base cache · ~1 GB free |
| **Time** | ~25 min active, ~30 min waiting (staged build) |
| **Difficulty** | ★★★ |
| **New concepts** | staging, recipe, local component, package-stack, import-deb |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | stages the build, packages and imports the stack |

## Before you start

- [ ] Tutorial 6 complete — `repos/` pinned, build toolchain installed from the cache
- [ ] The graybeard base cache from Tutorial 5 (it holds the toolchain and everything the generated packages depend on)
- [ ] ~1 GB free for `staging/` and the generated artifacts

## Milestone 1 — Stage the build

The in-place build of Tutorial 6 installed each component into `/usr`. For packaging, each component needs its own directory holding *exactly* its payload — the future `.deb`'s contents. The build script does this with `--stage DIR`: it installs into `staging/<component>` instead of `/usr`, and needs no root for that part.

> **Say what? — "staging":** a DESTDIR-style install root. `staging/hyprland-protocols/` contains precisely what the `hyprland-protocols` package will ship, and so on for all thirteen components.

### Step 1.1 — Rebuild into the staging root

The build dirs from Tutorial 6 are still warm, so most of the run reports "no work to do" — the work is redirecting each install into `staging/`:

```bash
vit@bis OpenFromSource$ sudo ./scripts/build-hyprland.sh --stage staging
```

<details><summary>staged build log (trimmed)</summary>

```console
vit@bis OpenFromSource$ sudo ./scripts/build-hyprland.sh --stage staging
[sudo] password for vit: 
==> [hyprland-protocols] meson setup
Directory already configured.

Just run your build command (e.g. ninja) and Meson will regenerate as necessary.
Run "meson setup --reconfigure" to force Meson to regenerate.

If build failures persist, run "meson setup --wipe" to rebuild from scratch
using the same options as passed when configuring the build.
==> [hyprland-protocols] ninja build
ninja: Entering directory `repos/hyprland-protocols/build'
ninja: no work to do.
==> [hyprland-protocols] install (staged to /home/vit/code/OpenFromSource/staging/hyprland-protocols)
ninja: Entering directory `repos/hyprland-protocols/build'
[0/1] Installing files
Installing /home/vit/code/OpenFromSource/repos/hyprland-protocols/protocols/hyprland-ctm-control-v1.xml to /home/vit/code/OpenFromSource/staging/hyprland-protocols/usr/share/hyprland-protocols/protocols
Installing /home/vit/code/OpenFromSource/repos/hyprland-protocols/protocols/hyprland-focus-grab-v1.xml to /home/vit/code/OpenFromSource/staging/hyprland-protocols/usr/share/hyprland-protocols/protocols
<SNIP — the staged install of all 13 components, in build order>
==> [sdbus-c++] cmake configure (source-built dependency 2.0.0)
-- 
--   *******************************************
--   *         SDBUS-C++ CONFIGURATION         *
--   *******************************************
-- 
--   SDBUSCPP_BUILD_LIBSYSTEMD: OFF
--   SDBUSCPP_INSTALL: ON
--   SDBUSCPP_BUILD_TESTS: OFF
--   SDBUSCPP_BUILD_CODEGEN: OFF
--   SDBUSCPP_BUILD_EXAMPLES: OFF
--   SDBUSCPP_BUILD_DOCS: ON
--     SDBUSCPP_BUILD_DOXYGEN_DOCS: OFF
--   BUILD_SHARED_LIBS: ON
-- 
-- Building with libsystemd v257
-- Building with documentation
-- Could NOT find Doxygen (missing: DOXYGEN_EXECUTABLE) 
-- Configuring done (0.0s)
-- Generating done (0.0s)
-- Build files have been written to: /home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/subprojects/sdbus-cpp/build
==> [sdbus-c++] cmake build
[ 90%] Built target sdbus-c++-objlib
[100%] Built target sdbus-c++
==> [sdbus-c++] install (staged to /home/vit/code/OpenFromSource/staging/sdbus-c++)
<SNIP — install lines omitted>
==> [xdg-desktop-portal-hyprland] install (staged to /home/vit/code/OpenFromSource/staging/xdg-desktop-portal-hyprland)
ninja: Entering directory `repos/xdg-desktop-portal-hyprland/build'
[0/1] Installing files
Installing src/xdg-desktop-portal-hyprland to /home/vit/code/OpenFromSource/staging/xdg-desktop-portal-hyprland/usr/libexec
Installing hyprland-share-picker/hyprland-share-picker to /home/vit/code/OpenFromSource/staging/xdg-desktop-portal-hyprland/usr/bin
Installing /home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/build/org.freedesktop.impl.portal.desktop.hyprland.service to /home/vit/code/OpenFromSource/staging/xdg-desktop-portal-hyprland/usr/share/dbus-1/services
Installing /home/vit/code/OpenFromSource/repos/xdg-desktop-portal-hyprland/hyprland.portal to /home/vit/code/OpenFromSource/staging/xdg-desktop-portal-hyprland/usr/share/xdg-desktop-portal/portals
build complete: the Hyprland stack is staged at /home/vit/code/OpenFromSource/staging/<component>
```

> **It's fine** — the `Directory already configured` / `ninja: no work to do` lines. The build dirs are reused from Tutorial 6; only the installs are redirected into `staging/`.

</details>

### Step 1.2 — Check the staging root

```bash
vit@bis OpenFromSource$ ls staging
aquamarine  hyprgraphics  Hyprland            hyprlang  hyprpaper  hyprwayland-scanner  xdg-desktop-portal-hyprland
hyprcursor  hypridle      hyprland-protocols  hyprlock  hyprutils  sdbus-c++
```

Thirteen directories, one per component — each one the payload of the package about to be built from it.

## Milestone 2 — Package and import the stack

Now the generic tool does the packaging. `package-stack` reads the **recipe**, and for each component in dependency order it builds a deterministic `.deb` from the staged directory (deriving `Depends` from the binaries' linked libraries via `dpkg-shlibdeps`), generates the complete rebuildable Debian source package (`.dsc` + referenced files), and imports the pair into the cache's `local` component — database first, verified only after the copy checksums, rolled back atomically on any failure.

> **Say what? — "recipe":** `recipes/hyprland.toml` — the entire stack-specific input: the 13 components, versions, build systems, metadata. The tool itself contains no stack knowledge.

> **Say what? — "local component":** the part of the cache holding locally built packages. It ships to operators through the ordinary handoff/sync, signed and hashed in the same `Release` as the archive components (read: "local" sits next to "main", "contrib", "non-free", "non-free-firmware").

> **Good to know** — To give a better indication of the progress we're making over the following steps, I bumped log level to INFO in `project.local.toml`.

### Step 2.1 — First try: the gate finds what the cache lacks

```bash
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
```
Output

```console
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
INFO:package_mgmt.lib.service.import_local:imported hyprland-protocols 0.6.4-1 (all) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprutils 0.7.1-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprwayland-scanner 0.4.4-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libsdbus-c++ 2.0.0-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libaquamarine 0.8.0-1 (amd64) into local component (verified)
{"error": "unsatisfiable dependency: libspng0 (>= 0.7.3) \u2014 not in cache", "exit_code": 1}
```

**An unsat dependency stops the import dead.** `libspng0` — the library behind hyprgraphics' image handling — is a real archive package, but it's not in the graybeard base set, so the tool refuses rather than ship a broken package. Add it to the cache and download it:

```bash
vit@bis OpenFromSource$ pm add libspng0
$ pm download
```

### Step 2.2 — Second try: one more gap

```bash
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
```
Output

```console
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
INFO:package_mgmt.lib.service.import_local:imported hyprland-protocols 0.6.4-1 (all) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprutils 0.7.1-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprwayland-scanner 0.4.4-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libsdbus-c++ 2.0.0-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libaquamarine 0.8.0-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprgraphics 0.1.3-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprlang 0.6.3-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprcursor 0.1.12-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hypridle 0.1.6-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprlock 0.8.2-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprpaper 0.7.5-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported xdg-desktop-portal-hyprland 1.3.9-1 (amd64) into local component (verified)
{"error": "unsatisfiable dependency: libxcb-errors0 (>= 1.0.1) \u2014 not in cache", "exit_code": 1}
```

Twelve imported, one more archive package missing — the `libxcb-errors0` runtime library for the XWayland error-handling API Hyprland uses. Add both the runtime and dev packages, and download:

```bash
vit@bis OpenFromSource$ pm add libxcb-errors-dev libxcb-errors0
$ pm download
```

## Checkpoint — third try's a charm

```bash
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
```
Output

```console
vit@bis OpenFromSource$ pm package-stack --recipe recipes/hyprland.toml
INFO:package_mgmt.lib.service.import_local:imported hyprland-protocols 0.6.4-1 (all) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprutils 0.7.1-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprwayland-scanner 0.4.4-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libsdbus-c++ 2.0.0-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libaquamarine 0.8.0-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprgraphics 0.1.3-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprlang 0.6.3-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported libhyprcursor 0.1.12-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hypridle 0.1.6-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprlock 0.8.2-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprpaper 0.7.5-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported xdg-desktop-portal-hyprland 1.3.9-1 (amd64) into local component (verified)
INFO:package_mgmt.lib.service.import_local:imported hyprland 0.49.0-1 (amd64) into local component (verified)
stack: hyprland
order: hyprland-protocols → hyprutils → hyprwayland-scanner → sdbus-c++ → aquamarine → hyprgraphics → hyprlang → hyprcursor → hypridle → hyprlock → hyprpaper → xdg-desktop-portal-hyprland → Hyprland
components imported: 13/13
Depends derived for 13 binaries (explicit deps merged; 23 unattributed libraries)
whole-set verify: PASS (no unlinked binaries, no unverified records)
simulated offline apt install: PASS (0 unmet dependencies)
```

**13/13 imported, whole-set verify PASS, and a simulated offline `apt install` of the whole set against a scratch mirror — PASS, 0 unmet dependencies.** The stack is in the cache: every binary verified, every binary linked (`binary_to_source`) to an imported, verified, rebuildable source.

> **It's fine** — the run re-lists the twelve already-imported components. `package-stack` is idempotent: it re-imports the identical pair once, no duplicate rows. The new line is the one that advances.

## Milestone 3 — Ship it to the operators

The `local` component flows through the ordinary handoff/sync channel unchanged. Sync a diff for the operators (ex-box from Tutorial 1, e-waste from Tutorial 4 — or handoff for a fresh box):

```bash
vit@bis OpenFromSource$ pm sync --system ex-box
```

Copy the `diff` onto the stick as usual. On the operator side, apply the diff, then re-run `bootstrap.sh` (idempotent) for it to write the `deb-src` line for `local`, so operators can fetch source offline too:

```bash
op@ex-box cache$ sudo cp -r diff /var/cache/package_mgmt/binary/
$ sudo /var/cache/package_mgmt/binary/diff/apply.sh
$ sudo /var/cache/package_mgmt/binary/handoff/bootstrap.sh
$ sudo apt update
```

Then the payoff — Hyprland, installed like any other package, entirely from the cache:

```bash
op@ex-box ~$ sudo apt install hyprland hypridle hyprlock hyprpaper xdg-desktop-portal-hyprland
```

You should see the whole stack install with **Download size: 0 B**. And because the source packages shipped too:

```bash
op@ex-box ~$ apt-get source hyprlang
```

…resolves offline from `local/source/Sources` and unpacks with `dpkg-source -x` — the same sources, at the same commits, that built the binaries. From Tutorial 1's single `vifm` to the full compositor stack, everything an offline box consumes now comes from a cache you built, signed, and verified yourself.

> **Good to know** — the `23 unattributed libraries` in the report are the archive libraries (libc, libwayland, libdrm, …) that the imported packages link against; they resolve from the frozen trixie cache, so nothing is silently dropped. If a library had *no* provider in the cache, the import would have refused — same as `libspng0` above.

## Milestone 4 — Bring a one-off rebuild into the cache too

`package-stack` handles whole stacks. For the ad-hoc rebuilds you do on an operator — like the `libxkbcommon-x11-0` you rebuilt in Tutorial 3 — there's `import-deb`, and it's what makes that fix permanent instead of a one-off on one box. Import a locally built `.deb` with its source:

```bash
vit@bis OpenFromSource$ pm import-deb libxkbcommon-x11-0_1.7.0-2_amd64.deb --source .
```

> **Say what? — "import-deb":** import a locally built `.deb` into the `local` component, linked to its source. The `--source <dir>` argument points at the directory holding the `.dsc` set — for a `dpkg-buildpackage` tree, that's the parent of the build directory, which already holds everything.

> **Watch out** — provenance: the `.dsc` in that parent is the *unmodified* source, but your `.deb` was built from the *patched* tree. If the recorded link must reproduce your binary, regenerate the source artifacts from the patched tree first — `dpkg-source -b .` (or `dpkg-buildpackage -S`), after bumping `debian/changelog` (e.g. `1.7.0-2+local1`) so the regenerated version is distinct.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `package-stack` aborts: "unsatisfiable dependency … not in cache" | A real archive package the stack links is missing from the cache | `pm add <pkg>` + `pm download`, then re-run — it's the gate working, exactly as shown |
| Same error repeatedly | You added the `-dev` but the runtime lib is also required | Add both (e.g. `libxcb-errors-dev libxcb-errors0`) |
| Staged build reinstalls everything | `staging/` empty or `--stage` path wrong | Confirm `staging/<component>` exists before `package-stack` |
| Operator can't install the stack | Diff not applied, or handoff not rebuilt | Re-run `apply.sh` then `bootstrap.sh` (idempotent), then `apt update` |
| `apt-get source` fails on the operator | The `deb-src` line for `local` missing | Re-run `bootstrap.sh` — it writes the sources entry |
| An imported package reports unverified | The `.deb` copy/checksum failed mid-import | Re-run `package-stack`; the import rolls back atomically on failure |

## Decoder ring

- **staging** — a DESTDIR-style install root; `staging/<component>` is exactly one package's payload
- **recipe** — `recipes/hyprland.toml`, the declarative per-stack input for `package-stack`
- **local component** — the cache partition holding locally built packages, shipped and signed like any archive component
- **package-stack** — package a staged build into `.deb` + `.dsc` pairs, import them, and validate the whole set
- **import-deb** — import a single locally built `.deb` into the `local` component, linked to its source

## Next

?

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*
