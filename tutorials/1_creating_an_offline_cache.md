# Tutorial 1 — Creating an offline cache to hand over on a USB stick

◂ [All tutorials](README.md) · [Tutorial 2 — A newer kernel from backports](2_kernel_from_backports.md) ▸

Let's build a local cache with packages, so that `apt` can install them on offline systems. 

> **Say what? — "the cache":** a folder tree of `.deb` files plus an index, built by `package_mgmt` and signed with its own GPG key so `apt` will trust it. From here on it's just "the cache".

The pitch: do your homework once, get everything you need into the cache, and then tear down and rebuild systems as you see fit. All with consistent behavior and without requiring additional internet access.

## At a glance

| | |
|---|---|
| **You'll end up with** | An offline Debian box that installs packages from a signed cache on a USB stick, zero bytes over the network |
| **You need** | Admin box with internet · offline box with a fresh Debian trixie install · USB stick (ext4 or btrfs) |
| **Time** | ~25 min active, ~5 min waiting (downloads) |
| **Difficulty** | ★☆☆ |
| **New concepts** | alias, virtual environment, profile, cache, index, archive sources, resolved, verified, moniker, handoff, sources list |

## The cast

| Who | Machine | Prompt | Role in this story |
|---|---|---|---|
| **Admin** | bis (online) | `vit@bis OpenFromSource$` | fills the cache, signs the payload |
| **Operator** | ex-box (offline) | `op@ex-box ~$` | consumes the payload, no network |

## Before you start

- [ ] Admin is on a Debian 13 (trixie) box with internet access
- [ ] Operator box is a fresh Debian trixie install, with SSH server and standard system utilities
- [ ] USB stick formatted **ext4 or btrfs** — symlinks required, FAT will not work
- [ ] You can become root on both machines

The recorded runs used the Debian 13.7 DVD installer. Where a command needs root, it's shown with `sudo`.

## Milestone 1 — Install the tool

`package_mgmt` is a Python program, and there are two ways to run it. The quick way — straight from its source tree — works, but it needs the checkout and a fresh alias every shell. This chapter does it the lasting way, the way every later chapter expects: install the tool into a *virtual environment* so `pm` is a real, durable command. 

> **Say what? — "virtual environment":** an isolated Python setup in `.venv` — the tool and its libraries live there, never touching the system Python. Its programs land in `.venv/bin/`.

### Step 1.1 — Get the project and its system dependencies

```bash
vit@bis $ git clone https://github.com/ingenerics/OpenFromSource
$ cd OpenFromSource
$ sudo apt update
$ sudo apt install -y git python3 python3-pip gpgv
```

We're working from the repo directory, so the prompt `vit@bis OpenFromSource$` is where we'll spend the rest of the admin's part.

### Step 1.2 — Install uv

It's best practice to run Python programs in an isolated environment. To be fair, this is less about security, more about not cluttering your system libraries. These tutorials create that environment with `uv`:

```bash
vit@bis OpenFromSource$ curl -LsSf https://astral.sh/uv/install.sh | sh
```
Output

```console
vit@bis OpenFromSource$ curl -LsSf https://astral.sh/uv/install.sh | sh
downloading uv 0.12.21 x86_64-unknown-linux-gnu
installing to /home/vit/.local/bin
  uv
  uvx
everything's installed!

To add $HOME/.local/bin to your PATH, either restart your shell or run:

    source $HOME/.local/bin/env (sh, bash, zsh)
    source $HOME/.local/bin/env.fish (fish)
```

### Step 1.3 — Create a virtual environment

Create it with a pinned Python, then activate it (note how the prompt changes):

```bash
vit@bis OpenFromSource$ uv venv --python 3.13.5 --seed --managed-python
```
Output

```console
vit@bis OpenFromSource$ uv venv --python 3.13.5 --seed --managed-python
Using CPython 3.13.5
Creating virtual environment with seed packages at: .venv
 + pip==26.2.1
Activate with: source .venv/bin/activate
```
Prompt change

```bash
vit@bis OpenFromSource$ source .venv/bin/activate
(OpenFromSource) vit@bis OpenFromSource$ 
```

### Step 1.4 — Install package_mgmt into the venv

Unless your last name is Thompson or Ritchie, a two-letter program name is asking for trouble. In the activated venv `pip install .` gives you `package_mgmt` (and pulls `pycdlib`, the ISO reader that Tutorial 5's `ensure-base` needs):

```bash
(OpenFromSource) vit@bis OpenFromSource$ pip install .
```
Output

```console
(OpenFromSource) vit@bis OpenFromSource$ pip install .
Processing ./.
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Preparing metadata (pyproject.toml) ... done
Collecting python-debian (from package_mgmt==0.1.0)
  Downloading python_debian-1.1.1-py3-none-any.whl.metadata (2.6 kB)
Collecting pycdlib (from package_mgmt==0.1.0)
  Downloading pycdlib-1.21.0-py3-none-any.whl.metadata (1.3 kB)
Downloading pycdlib-1.21.0-py3-none-any.whl (237 kB)
Downloading python_debian-1.1.1-py3-none-any.whl (138 kB)
Building wheels for collected packages: package_mgmt
  Building wheel for package_mgmt (pyproject.toml) ... done
  Created wheel for package_mgmt: filename=package_mgmt-0.1.0-py3-none-any.whl size=209942 sha256=7ebd57a4926c914da80cdaf3e23d13aacdfb345120ac3a2a319d050c4cc8949f
  Stored in directory: /tmp/pip-ephem-wheel-cache-3u6b9ku9/wheels/1b/3a/a3/aa677c149867b2325985865222dccbbed48c17d7d83ca8f3d3
Successfully built package_mgmt
Successfully installed package_mgmt-0.1.0 pycdlib-1.21.0 python-debian-1.1.1
```

### Step 1.5 — Add the `pm` alias

Once we have the program on our system, we can do what we want of course. Let's add an alias to save some keystrokes.  Check where the binary landed, then put the alias in `~/.bashrc` so it survives every shell:

```bash
(OpenFromSource) vit@bis OpenFromSource$ which package_mgmt
/home/vit/code/OpenFromSource/.venv/bin/package_mgmt
```
Use your favorite editor to edit `~/.bashrc`.

```bash
# package_mgmt
alias pm=/home/vit/code/OpenFromSource/.venv/bin/package_mgmt
```

> **Say what? — "alias":** a shell shorthand. `alias pm='…'` means that from now on, when you type `pm`, the shell runs whatever the alias points to. Here it points at the executable within the virtual environment — `.venv/bin/package_mgmt` — so `pm` is that program by a shorter name. You can activate the alias by executing `source ~/.bashrc` or by opening a new shell.

> **It's fine** — sourcing `~/.bashrc` kicks you out of the virtual environment. It doesn't matter: the alias calls the binary inside `.venv` directly, and that uses the venv's own interpreter — you don't need the venv active to run `pm`. You *do* need to be at the project root, because that's where the TOML configs live (`project.defaults.toml`, the shared baseline — and `project.local.toml`, your git-ignored overlay).

The alias works from anywhere:

```bash
vit@bis OpenFromSource$ pm --help
```
Output

```console
vit@bis OpenFromSource$ pm --help
usage: package_mgmt [-h]
                    {add,ensure-base,import-deb,init,download,config,list,show,verify,systems,snapshot,handoff,sync,align-repo,package-stack} ...

OpenFromSource package management CLI

positional arguments:
  {add,ensure-base,import-deb,init,download,config,list,show,verify,systems,snapshot,handoff,sync,align-repo,package-stack}
    add                 Resolve and register a package and its dependencies
    ensure-base         Register offline DVD installer image metadata in the cache
    import-deb          Import a locally built .deb into the local component
    init                Initialize cache directories and database
    download            Download and verify pending files into the cache pool
    …
    align-repo          Determine or verify the version-aligned source set (admin-only)
    package-stack       Package and import a source-built stack into the local component (admin-only)

options:
  -h, --help            show this help message and exit
```

…but commands that read the config fail away from the project root:

```bash
vit@bis ~$ pm init
```
Output

```console
vit@bis ~$ pm init
{"error": "Default config not found: project.defaults.toml", "exit_code": 102}
```

## Milestone 2 — Ensure the regular_user profile

A **profile** is a named set of `package_mgmt` settings: it determines how much of each package's sources and build-tools get fetched. Three ship with the project, and this tutorial needs the default one.

> **Say what? — "profile":** a named preset for how deep `package_mgmt` goes when it resolves packages — binaries only, binaries plus the sources of what you asked for, or everything with all build tools.

| Profile | Sources? | Source depth | What it's for | Used in |
|---|---|---|---|---|
| **regular_user** | no | 0 | install binaries from the cache, nothing more | this tutorial |
| **developer** | yes | finite (e.g. 1) | rebuild the packages you care about | Tutorial 3 |
| **graybeard** | yes | unlimited (`-1`) | full rebuild from source — turtles all the way down | Tutorials 5–7 |

`project.defaults.toml` already ships the regular_user presets (`source = false`, `max_source_dep_depth = 0`).

If you need different settings (multiple releases, extra components, or a different mirror), you can create an additional `project.local.toml` with overrides in the project root. We'll do so in the next tutorial.

> **Why a local override file?** `project.defaults.toml` is version-controlled and shipped with the project. Your local settings stay in `project.local.toml`, which is git-ignored.

Verify you're on regular_user by reading the effective config — `pm config` prints every setting with `(default)` or `(override: project.local.toml)`:

```bash
vit@bis OpenFromSource$ pm config
```
Output

```console
vit@bis OpenFromSource$ pm config
Configuration:
  base_dir              /var/cache/package_mgmt    (default)
  …
  fetch_source          false                      (default)
  …
  max_source_dep_depth  0                          (default)
```

`fetch_source: false` and `max_source_dep_depth: 0`, both `(default)` — that's regular_user. The deeper profiles flip these: Tutorial 2 adds a local component, Tutorial 3 turns on the developer profile, Tutorial 5 the graybeard one.

## Milestone 3 — Seed the cache on the admin box

Everything the offline box will ever consume has to exist here first. The admin box is online, so it can reach Debian's mirrors, download the packages, and verify them against the signed indexes before anything leaves the system.

### Step 3.1 — Create the cache directory

`package_mgmt` keeps its state under `/var/cache/package_mgmt`. Create it and hand it to your user:

```bash
$ export CACHEPATH="/var/cache/package_mgmt"
$ sudo mkdir $CACHEPATH
$ sudo chown -R $USER:$USER $CACHEPATH
```

### Step 3.2 — Initialize the cache

`pm init` builds the database and fetches the package **indexes** from the archive sources configured in the profile. It can take a couple of minutes.

```bash
vit@bis OpenFromSource$ pm init
```

> **Say what? — "index":** the machine-readable list of every package and version in a repository. `pm init` downloads those lists (the `Packages.gz` files) and builds its database from them, so the resolver can search without hammering the network. It also records *provides* — virtual names one package offers (like `pkg-config`, provided by `pkgconf`) — which is what the log's `69982 provides entries` line means.

> **Say what? — "archive sources":** the Debian repositories `package_mgmt` mirrors. This is where the resolver looks for packages. For the log below we used a profile listing `trixie` main/contrib/non-free plus `trixie-backports` main.

<details><summary>Full init log</summary>

```console
INFO:package_mgmt.lib.service.initializer:Binary cache /var/cache/package_mgmt/binary (was created)
INFO:package_mgmt.lib.service.initializer:Source cache /var/cache/package_mgmt/source (was created)
INFO:package_mgmt.lib.service.db_init:Database created at /var/cache/package_mgmt/package_mgmt.db (schema v1)
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/main/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/contrib/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie/non-free/binary-amd64/Packages.gz
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.gz
WARNING:package_mgmt.cli.commands.init:Failed to fetch https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.gz: index main/binary-amd64/Packages.gz is not listed in the signed release at https://deb.debian.org/debian/dists/trixie-backports
INFO:package_mgmt.cli.commands.init:Fetching: https://deb.debian.org/debian/dists/trixie-backports/main/binary-amd64/Packages.xz
INFO:package_mgmt.lib.service.initializer:Populated 69982 provides entries into database
Base directory:    /var/cache/package_mgmt
Binary cache:      /var/cache/package_mgmt/binary
Source cache:      /var/cache/package_mgmt/source
Database:          /var/cache/package_mgmt/package_mgmt.db
Schema version:    1
Default source:    trixie/main
Archive sources:
  [1] trixie/main — 68825 binaries
  [2] trixie/contrib — 303 binaries
  [3] trixie/non-free — 741 binaries
  [4] trixie-backports/main — 3279 binaries
Provides entries:  69982
Status:            initialized
INFO:package_mgmt.cli.commands.init:Init completed (base_dir=/var/cache/package_mgmt, base_dir=/var/cache/package_mgmt, schema=v1, provides=69982, status=initialized)
```

> **It's fine** — the `WARNING` about `Packages.gz` "not listed in the signed release" for backports. Debian publishes the backports index compressed as `.xz`; `package_mgmt` noticed, fetched the `.xz` instead, and carried on.

</details>

### Step 3.3 — Register the first packages

Create a package list file. This is the shopping list for the whole box: a terminal, an editor, a file manager, monitors, a firewall, and storage tools.

```txt
# base console tools
kitty
neovim
vifm
btop
nvtop

# firewall
ufw

# storage management
ntfs-3g
libnfs-utils
```

Feed it to the resolver:

```bash
vit@bis OpenFromSource$ pm add --file /tmp/base_pkgs.txt
```

You should get something like:

```console
INFO:package_mgmt.lib.service.resolver:Indexes loaded: 73148 binary packages, 0 source packages (2 sources configured)
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: libgcc-s1 -> libc6
WARNING:package_mgmt.lib.service.resolver:Circular dependency detected: openssl-provider-legacy -> libssl3t64
Resolved 8/8
Packages requested:  8
Packages resolved:   8
Errors:              0
```

> **It's fine** — the two `Circular dependency detected` warnings. `libgcc-s1 -> libc6` and friends are real loops in Debian's graph; the resolver knows how to break them. Keep an eye on `Errors:` at the end of the summary instead.

> **Say what? — "resolved":** the resolver worked out the full set of packages and exact versions needed to satisfy your list, including every dependency — the "closing the package graph" step.

### Step 3.4 — Verify the graph is complete

Look at what `vifm` drags in, then run the full verification:

```bash
vit@bis OpenFromSource$ pm list --deps vifm
$ pm verify
```

`pm list --deps vifm` prints vifm's dependency closure. `pm verify` checks the whole graph and reports two sections: runtime dependencies and build-tool dependencies. 

<details><summary>Full verification log</summary>

The long `Beyond source depth` list is the closure of every package that has landed in the cache.

```console
Dependency Graph Verification
─────────────────────────────
Total packages:         157
Total binaries:         156
Total sources:          0
Unresolved deps:        0
Unlinked binaries:      0
Beyond source depth:    156 (informational)
  - kitty (= 0.41.1-2+deb13u2)
  - kitty-shell-integration (= 0.41.1-2+deb13u2)
  - kitty-terminfo (= 0.41.1-2+deb13u2)
  - python3 (= 3.13.5-1)
<SNIP — 150 lines omitted>
  - libnfs14 (= 5.0.2-1+deb13u1)
Circular dependencies:  2
  - libgcc-s1 -> libc6 (circular)
  - openssl-provider-legacy -> libssl3t64 (circular)
Runtime Status:         COMPLETE

Build-Tool Dependencies
───────────────────────
Total build-dep links:  0
Total build tools:      0
Unresolved build deps:  0
Missing build tools:    0
Build Status:           COMPLETE

Status:                 COMPLETE
```

> **It's fine** — `Beyond source depth: 156 (informational)`. You asked for binaries only, so the resolver pins every package to its exact binary version. It's reporting, not complaining.

</details>

### Step 3.5 — Download the payload

This might come as a surprise, but all we've done so far is `add` metadata to the `package_mgmt` database. This design is intentional, not an afterthought. We can add as many packages as we'd like *before* triggering the download. Which will then only pull the minimal set that has not been retrieved yet.

Now pull down every file in the graph and verify each one against the signed index:

```bash
vit@bis OpenFromSource$ pm download
```

You should see a progress bar and a summary:

```console
vit@bis OpenFromSource$ pm download
Elapsed: 00:04:05
[████████████████████████████████████████] 156/156 files (74.2 MiB/74.2 MiB)
…
Total: 156 files downloaded, 156 verified, 0 failed (00:04:05)
```

> **Say what? — "verified":** each downloaded file's checksum was checked against the signed index before being accepted into the cache. A file that fails verification is never installed on anyone's box.

The cache now holds 156 binary files — 74.2 MiB of packages that can be installed with no network at all.

## Milestone 4 — Give the box a name, pack the payload

The cache is built, but `apt` on the operator box needs a *signed* repository to trust. `package_mgmt` names each consumer machine and generates a handoff: a snapshot of the cache packaged as a minimal signed apt repository.

### Step 4.1 — Register the operator machine

Every box that receives a handoff needs a **moniker**. Let's create one for our purely fictional example box:

```bash
vit@bis OpenFromSource$ pm systems add ex-box
```

> **Say what? — "moniker":** the name `package_mgmt` uses to track which snapshot belongs to which consumer box. Snapshots and syncs are always addressed by moniker.

### Step 4.2 — Generate the handoff

```bash
$ pm handoff --system ex-box
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
Snapshot:       156 binaries marked for 'ex-box'
```

> **Say what? — "handoff":** a self-contained snapshot of the cache, arranged as an apt repository and signed with the cache's own GPG key, together with a `bootstrap.sh` that wires it into apt. This is what travels on the stick.

### Step 4.3 — Copy the cache to the stick

Mount the stick and copy the whole `package_mgmt` directory onto it. The stick must be **ext4 or btrfs**: the cache links files as `pool → ../pool` symlinks, and FAT has no symlinks. 

<details>
<summary>Key-handling posture</summary>

The handoff payload itself carries no secret key material (only the public `offline-cache.gpg`). However, the handoff references the cache above it (its `pool`/`source` links point back into `package_mgmt/binary/`). This means the transported unit is the whole cache root, so a plain copy of `package_mgmt/` also carries the keyring (which lives at `package_mgmt/.gnupg-handoff/`). Exclude `.gnupg-handoff/` from operator media unless the operator is a trusted relay that must re-sign.

</details>

## Milestone 5 — Bootstrap the offline box

The stick arrives at the offline box. From here on the operator is on their own: no internet, no admin, just the stick.

### Step 5.1 — Check the stick and copy the cache in

```bash
op@ex-box flash$ ls -lah
```

You should see the `package_mgmt` directory, then copy it to the system cache path:

```bash
op@ex-box flash$ sudo cp -r package_mgmt /var/cache/
```

> **Say what? — "the cache" revisited:** same thing as on the admin side — now it's the copy at `/var/cache/package_mgmt` on the offline box. Apt will be pointed at the `handoff` subdirectory.

### Step 5.2 — Run the bootstrap

```bash
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/handoff/bootstrap.sh
```

<details><summary>Full bootstrap log</summary>

```console
op@ex-box ~$ sudo /var/cache/package_mgmt/binary/handoff/bootstrap.sh
Ign:1 file:/var/cache/package_mgmt/binary/handoff trixie InRelease
Ign:2 file:/var/cache/package_mgmt/binary/handoff trixie-backports InRelease
Get:3 file:/var/cache/package_mgmt/binary/handoff trixie Release [2,991 B]
Get:3 file:/var/cache/package_mgmt/binary/handoff trixie Release [2,991 B]
Get:4 file:/var/cache/package_mgmt/binary/handoff trixie-backports Release [1,122 B]
Get:5 file:/var/cache/package_mgmt/binary/handoff trixie Release.gpg [488 B]
Get:4 file:/var/cache/package_mgmt/binary/handoff trixie-backports Release [1,122 B]
Get:6 file:/var/cache/package_mgmt/binary/handoff trixie-backports Release.gpg [488 B]
Get:5 file:/var/cache/package_mgmt/binary/handoff trixie Release.gpg [488 B]
Get:6 file:/var/cache/package_mgmt/binary/handoff trixie-backports Release.gpg [488 B]
Get:7 file:/var/cache/package_mgmt/binary/handoff trixie/main amd64 Packages [14.2 kB]
101 packages can be upgraded. Run 'apt list --upgradable' to see them.
Bootstrap: configuring apt from /var/cache/package_mgmt/binary/handoff (releases: trixie, trixie-backports)
Source added: /etc/apt/sources.list.d/openfromsource-cache.list
apt update: OK
```

> **It's fine** — the `Ign:` lines and the repeated `Get:` lines for `Release`. The `Ign:` entries are apt skipping unsigned `InRelease` files (the signed ones come next), and the repeats are apt's parallel fetch interleaving. Both are routine.

The important lines are the last two: the sources file was created and `apt update` succeeded.

</details>

### Step 5.3 — See the sources apt now trusts

```bash
op@ex-box ~$ cat /etc/apt/sources.list.d/openfromsource-cache.list
```
Output

```console
op@ex-box ~$ cat /etc/apt/sources.list.d/openfromsource-cache.list
deb [signed-by=/var/cache/package_mgmt/binary/handoff/offline-cache.gpg] file:/var/cache/package_mgmt/binary/handoff trixie main contrib non-free
deb [signed-by=/var/cache/package_mgmt/binary/handoff/offline-cache.gpg] file:/var/cache/package_mgmt/binary/handoff trixie-backports main
```

> **Say what? — "sources list":** the file that tells apt where to look for packages. Here both entries point at the local handoff directory on the stick's copy, signed by the cache's key. There is no `http://` anywhere.

### Step 5.4 — Upgrade the fresh install from the cache

The installer's media is older than the cache you just built, so the box is behind by 101 packages. Bring it up to date:

```bash
op@ex-box ~$ sudo apt update
$ sudo apt upgrade
```

You should see the plan — and the important number:

```console
op@ex-box ~$ sudo apt upgrade
Upgrading:
  adduser  libcrypt1  …  zlib1g
…
Summary:
  Upgrading: 101, Installing: 0, Removing: 0, Not Upgrading: 0
  Download size: 0 B / 41.9 MB
  Freed space: 183 MB

Continue? [Y/n]
```

**Download size: 0 B.** Every one of those 101 packages is served from the local handoff — the offline box is now its own mirror. Answer `Y` and let it unpack.

## Checkpoint — the box is served, not downloading

The moment of truth: install something and watch apt never touch the network. `vifm` is a nice test because the base install didn't include it, so it has to come entirely from the cache:

```console
op@ex-box ~$ sudo apt install vifm
Installing:
  vifm

Installing dependencies:
  libgpm2  libsodium23  vim  vim-runtime

Summary:
  Upgrading: 0, Installing: 5, Removing: 0, Not Upgrading: 101
  Download size: 0 B / 9,983 kB
  Space needed: 0 B / 470 GB available

Continue? [Y/n]
```

Five packages installed, 9,983 kB of payload, **zero bytes downloaded**. That's the whole point: an offline box that runs `apt install` as if it had a mirror in the room.

To complete the tutorial, you can install the rest of the set:

```bash
op@ex-box ~$ sudo apt install kitty neovim vifm btop nvtop \
    ufw ntfs-3g libnfs-utils
```

If you are executing the operator part through SSH, you can reboot (or restart your shell) so kitty's SSH-side benefits kick in.

> **Good to know** — if you keep relevant dotfiles in a backed-up home folder, now is the time to copy them over.

## If it breaks

| Symptom | Probably means | Do this |
|---|---|---|
| `pm init` fetches forever | Big index downloads on a slow link | Let it finish; it's a one-time cost. Check `Archive sources` counts at the end |
| `Errors:` is not 0 after `pm add` | A requested package or dependency couldn't be resolved | Check the package name and that the right archive source is configured |
| Stick shows 0 B or missing files on the operator box | FAT/exFAT stick, or the copy was interrupted | Reformat ext4 (or btrfs) and re-copy; the cache needs symlinks |
| `bootstrap.sh` says `apt update` failed | Cache path differs, or the copy is incomplete | Verify `/var/cache/package_mgmt/binary/handoff` exists with the `.deb`s |
| `apt install` wants to download from the internet | Sources still point at Debian mirrors, not the handoff | Check `openfromsource-cache.list` and that the handoff directory is in place |

## Decoder ring

- **alias** — a shell shorthand: typing `pm` runs the executable the alias points to (here, the one the virtual environment provides)
- **virtual environment** — an isolated `.venv` folder holding a Python program and its libraries; the program's executable lands in `.venv/bin/`
- **profile** — a named preset for how deep `package_mgmt` goes when resolving packages (regular_user / developer / graybeard)
- **the cache** — a folder tree of `.deb` files plus an index, built by `package_mgmt` and signed with its own GPG key so apt trusts it
- **index** — the machine-readable list of every package and version in a repository
- **archive sources** — the Debian repositories `package_mgmt` mirrors and searches
- **resolved** — the resolver worked out the exact set and versions of packages, including dependencies
- **verified** — each downloaded file's checksum was checked against the signed index
- **moniker** — the name `package_mgmt` uses to track snapshots per consumer box
- **handoff** — a self-contained, signed snapshot of the cache plus a `bootstrap.sh` that wires it into apt
- **sources list** — the file that tells apt where to look for packages

## Next

[Tutorial 2 — A newer kernel from backports](2_kernel_from_backports.md)

---

*Tested against OpenFromSource alpha (@ 2026-10-20) on Debian trixie; commands and logs are from the recorded runs, trimmed only where marked.*