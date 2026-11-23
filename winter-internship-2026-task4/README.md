
# eSim 2.6 on Ubuntu 25.04: installation failure, root cause, fix

FOSSEE Winter Internship 2026, Screening Task 4 · Sridaya T M · Chennai Institute of Technology · tested 20-23 November 2026

---

**eSim 2.6 cannot be installed on Ubuntu 25.04.** The downloads page lists 25.04
as supported. `install-eSim.sh --install` aborts at step 7 of 11 and leaves
nothing usable. The cause is two lines in a dependency table.

| # | Issue | File | Severity | Status |
| --- | --- | --- | --- | --- |
| 1 | `libcanberra-gtk-module` has no candidate on 25.04, install aborts at 7/11 | `nghdl/install-nghdl.sh` | Blocks entire install | **Fixed**, 2-line patch, verified |
| 2 | `--install` cannot run without a terminal | `Ubuntu/install-eSim.sh` | Blocks CI and automated verification | **Fixed**, 87-line patch, verified |

Three smaller automation defects are listed under [Other observations](#other-observations).

## Verification chain, Issue 1

Each step reproducible. Logs linked.

| Step | Result | Evidence |
| --- | --- | --- |
| Download `eSim-2.6.zip` from esim.fossee.in | 86,783,803 bytes | |
| Verify SHA256 against the downloads page | match | `79ac33c3…4e5b8c` |
| Install on clean Ubuntu 25.04 | **aborts, step 7/11** | [`run-25.04-PUBLISHED-ZIP.log`](logs/run-25.04-PUBLISHED-ZIP.log) |
| Extract `nghdl.zip`, inspect the profile table | defect present at lines 117-121 | |
| Apply 2-line patch | | [`patches/fix-canberra-25.04.patch`](patches/fix-canberra-25.04.patch) |
| Re-install, same container, same command | **11/11, toolchain doctor passes** | [`run-25.04-PUBLISHED-ZIP-FIXED.log`](logs/run-25.04-PUBLISHED-ZIP-FIXED.log) |

The bug is in the bytes FOSSEE publishes, not only in the source tree.

## What was tested

| | |
| --- | --- |
| Artifact | `eSim-2.6.zip`, esim.fossee.in/downloads, 86,783,803 bytes |
| SHA256 | `79ac33c315b142308c078f00a4d4383335cadbdaef1d2841231acba3504e5b8c` |
| Matches published hash | yes |
| OS | `ubuntu:25.04` container, clean per run |
| Source references | `master` @ `d86c189`, tag `v2.6` |

The release ships `install-nghdl.sh` identical to `d86c189`, confirmed with
`diff`. Line numbers below apply to both the zip and the repository.

**Why 2.6 and not 2.5.** The task specifies eSim 2.5 on Ubuntu 25.04. The
downloads page lists 2.5 as supporting Ubuntu 22.04, 23.04 and 24.04 only. A
failure there would be an unsupported configuration, not a defect. 2.6 is the
current download and the first release to claim 25.04.

**Why containers and not VirtualBox.** Every run starts from an identical clean
state, which matters when attributing a failure to one cause. The defect is an
apt archive-level failure, shown below with a direct archive query. GUI runtime
was not tested; see [Limitations](#limitations).

---

# Issue 1: install aborts at step 7 on Ubuntu 25.04

### Symptom

```
>>> [7/11] Building NGHDL and ngspice
>>> NGHDL profile: Ubuntu 25.04 | CFLAGS=''
>>> Installing build dependencies
Package libcanberra-gtk-module is not available, but is referred to by another package.
E: Package 'libcanberra-gtk-module' has no installation candidate
Error! Kindly resolve the above error(s) and try again.
Aborting NGHDL installation...
=== Installation aborted ===
  Reason       NGHDL install failed (exit 100).
```

Steps 1-6 complete. KiCad 8 and the eSim symbol library install. Steps 7-11
never run: no ngspice, no SKY130 PDK, no launcher, no toolchain check. eSim
cannot start or simulate.

### Cause

`nghdl/install-nghdl.sh:89`

```
GTK_CANBERRA=""     # gtk2+gtk3 vs gtk3-only (libcanberra-gtk-module dropped in 26.04)
```

`detect_profile()`, lines 116-122:

```
case "$UBUNTU_VER" in
    26.04)
        GTK_CANBERRA="libcanberra-gtk3-module"
        ;;
    23.04|24.04|25.04)
        GTK_CANBERRA="libcanberra-gtk-module libcanberra-gtk3-module"
        ;;
```

The comment and the table both assume the GTK2 module disappears in 26.04. It
disappeared in 25.04.

| Release | `libcanberra-gtk-module` |
| --- | --- |
| noble (24.04) | published, binary of source package `libcanberra` |
| plucky (25.04) | no installation candidate |

Upstream: Launchpad #2111252 records the identical apt error on 25.04, with the
Ubuntu maintainer confirming the package is absent from plucky. It was removed
because GTK2 is deprecated (Debian #967569). Removed, not renamed. No
replacement binary exists.

25.04 sits one branch too high in the case statement.

### This is not a container artefact

The obvious objection is that `universe` was not enabled. A direct archive query
on the same release settles it:

```
$ apt-cache policy libcanberra-gtk-module libcanberra-gtk3-module
libcanberra-gtk-module:
  Installed: (none)
  Candidate: (none)
  Version table:
libcanberra-gtk3-module:
  Installed: (none)
  Candidate: 0.30-17ubuntu2
  Version table:
     0.30-17ubuntu2 500
        500 http://archive.ubuntu.com/ubuntu plucky/main amd64 Packages
```

Same archive, same command, same release. The gtk3 module resolves from
plucky/**main**, which needs no extra repository at all. The gtk2 module has no
candidate and an empty version table: apt knows the name because another package
still lists it as a dependency, but no version is published anywhere. That is
exactly what the installer's error reports when it says the package "is not
available, but is referred to by another package".

Universe was enabled regardless. Step 5 of the same failing run installed KiCad 8
from universe in 1m54s, which is what `detect_profile()` specifies for 25.04
(`KICAD_SOURCE="universe"`).

### Present in the published release

`eSim-2.6.zip` ships `nghdl.zip`, unpacked by the installer at step 7.
Extracted from the SHA256-verified download:

```
116:    case "$UBUNTU_VER" in
117-        26.04)
118-            GTK_CANBERRA="libcanberra-gtk3-module"   # gtk2 variant gone in 26.04
119-            ;;
120-        23.04|24.04|25.04)
121-            GTK_CANBERRA="libcanberra-gtk-module libcanberra-gtk3-module"
122-            ;;
```

### Fix

[`patches/fix-canberra-25.04.patch`](patches/fix-canberra-25.04.patch)

```
117c117
<         26.04)
>         26.04|25.04)
120c120
<         23.04|24.04|25.04)
>         23.04|24.04)
```

23.04 and 24.04 keep the GTK2 module, still published for those releases. 25.04
joins 26.04 on gtk3-only.

Suggested follow-up, not implemented here: query apt for an installation
candidate before requesting the GTK2 package. Two lines fix 25.04. A candidate
check stops the next release reproducing it.

### Verified

Same container, same command, patched:

```
>>> NGHDL post-install self-check
>>> code model present: ghdl.cm*
>>> code model present: ivlng*
>>> Self-check passed.
>>> NGHDL installed successfully on Ubuntu 25.04
...
All checks passed.
=== eSim 2.6 installed successfully ===
  Took         17m37s
EXIT=0
```

Step 11 is `runToolchainDoctor`, eSim's own verification. It marks `[OK]` for
ngspice-45, all eight code models including `ghdl.cm`, the ivlng adapter,
Icarus Verilog 14.0 with `vvp` and `libvvp`, Verilator 5.032 and GHDL 5.0.1.

The patched run emits no `[WARN]` lines. Every optional dependency the installer
guards with a warning resolved on 25.04.

### Prior report

Issue #637, open since 20 August 2026, reports this apt error as item 3 of four.
Two differences from the findings here.

It attributes the cause to the package being "obsolete/renamed under Ubuntu
24.04 repositories". The package is still published for noble, and it was
removed in plucky rather than renamed.

It was filed against a different tree. Its paths (`scripts/setup-esim.sh`,
`/3rdparty/template/sym-lib-table`, `ghdl-4.1.0.tar.gz`) do not exist in current
master, and the reporter states they were on branch `installer`.
`detect_profile()` did not exist in that tree.

The symptom is known. The location and cause in the current unified installer
are not.

---

# Issue 2: `--install` cannot run unattended

### Symptom

`read -rp` is called five times on the install path.

| Line | Prompt | Reached |
| --- | --- | --- |
| 1226 | proxy? (y/n) | every run, `setupProxy` |
| 1228-1231 | proxy host / port / user / password | only if proxy=y (1231 is `read -rsp`) |
| 1079 | install IHP Open PDK? (y/n) | every run, `installIhpPdk` |
| 801 | remove /usr/local GHDL? (y/n) | only if a stray GHDL exists |
| 897 | remove it and install KiCad >= N? (y/n) | only if an old KiCad exists |

`case "$option"` at line 1293 accepted `--dry-run`, `--install`, `--uninstall`.
No `--yes`. No environment override. With stdin at `/dev/null` the installer
dies at step 0 of 11.

### Why this reads as an oversight

Three things in the same codebase assume unattended runs work.

1. Lines 1314-1324 export `NEEDRESTART_MODE=l`, with a comment that needrestart
   "BLOCKS on a keypress, so an unattended install silently stalls, looking
   hung, for as long as nobody is watching".
2. `ui_step()` (375) and `ui_log()` (416) print `>>> [N/TOTAL] <label>` whenever
   `UI_LEVEL < 1`, which is any non-TTY run. A machine-readable format, on
   purpose.
3. `Ubuntu/bootstrap.sh` defines `require_tty()`, commented: "install/uninstall
   prompt for proxy, sudo password, IHP, etc. Under `curl | bash` stdin is the
   script pipe, so those reads must come from the terminal instead." Line 265
   runs `--install < /dev/tty`. Line 269 runs `--dry-run < /dev/null`.

The terminal dependency was identified and solved for interactive users.
`--dry-run` already runs without a terminal. `--install` did not.

### Four consequences, all reproduced

1. **A failing `read` kills the install.** At EOF `read` returns non-zero.
   `--install` runs under `set -e` with an ERR trap, so the installer exits.
2. **The error names the wrong step.** `ui_step` prints the label before calling
   the function, so a dead prompt inside `installIhpPdk` reported "Failed at
   step 9/11 Installing the IHP Open PDK". No PDK install was attempted.
3. **Piping answers does not work.** `printf 'n\nn\n' |` failed at step 9. The
   five-minute NGHDL build spawns children that inherit and consume the same
   stdin, so the second answer was gone before it was needed.
4. **debconf preconfiguration is skipped silently.** Non-TTY runs fall through
   Dialog, Readline and Teletype frontends, log `dpkg-preconfigure: unable to
   re-open stdin`, then continue. Nothing broke here. A package that genuinely
   needs preconfiguration would not get it.

### Fix

[`patches/fix-noninteractive-install.patch`](patches/fix-noninteractive-install.patch),
87 lines, six hunks, against `Ubuntu/install-eSim.sh`.

- `ask()` helper taking a variable name, environment override, default and
  prompt. Precedence: environment variable, then `--yes` default, then prompt as
  before.
- `--yes` / `-y` as a flag; `ESIM_ASSUME_YES=1` as an environment equivalent.
- The four install-path prompts converted to `ask` calls.
- Argument guard strips `--yes` before counting. Without this the existing
  `[ "$#" -ne 1 ]` check rejects `--install --yes` before the case statement
  runs.
- Usage text updated.

All defaults are `n`. Nothing is deleted, nothing optional is added silently.
Each decision is written to `eSim-install.log` as
`non-interactive: <prompt> -> n (default)`, so an unattended run records what
was chosen on the user's behalf.

Behaviour without the flag is unchanged.

### Verified

```
bash Ubuntu/install-eSim.sh --install --yes < /dev/null

>>> non-interactive: Is your internet connection behind a proxy? (y/n) -> n (default)
>>> Installing without proxy
>>> [1/11] Clearing artifacts of older eSim installs
...
All checks passed.
=== eSim 2.6 installed successfully ===
  Took         10m45s
EXIT=0
```

Same stdin condition that killed the unpatched installer at step 0. No terminal,
no piped answers, no human present. 11/11 and the doctor passes.

### Still not fully CI-ready

`--yes` handles eSim's own prompts. Two things remain, both under Other
observations: `sudo` is called unconditionally, and `DEBIAN_FRONTEND` does not
survive `sudo`. A complete non-interactive mode would resolve `sudo` once with a
root fallback and pass `DEBIAN_FRONTEND` through to the installer's apt calls.

---

## Other observations

Not defects for a desktop install. All three affect automated runs.

**`sudo` called unconditionally.** A root environment with no `sudo` binary fails
at line 833, `sudo: command not found`, exit 127. The usual pattern is to resolve
sudo once and fall back to empty when already root.
Evidence: [`repro-25.04-answered.log`](logs/repro-25.04-answered.log)

**`DEBIAN_FRONTEND` does not survive `sudo`.** sudoers defaults to `env_reset`,
so setting it before running the installer has no effect on its `sudo apt-get`
calls. Observed as tzdata's geographic-area menu blocking a container run. A user
who knows the correct variable still gets prompted.

**`test_install.sh` step counts disagree.** Its header says twelve install steps.
Its stub list covers thirteen functions. `INSTALL_STEPS` defines eleven numbered
steps plus two pre-flight. Not a bug. Three numbers for one thing.

## Reproduction

Docker only. No VM. Under twenty minutes.

```bash
wget https://static.fossee.in/esim/installation-files/eSim-2.6.zip
sha256sum eSim-2.6.zip   # 79ac33c3...4e5b8c
python3 -m zipfile -e eSim-2.6.zip .

# fails at step 7, ~6 min
yes n | docker run --rm -i -v "$PWD/eSim-2.6":/esim -e HOME=/logs \
  -e DEBIAN_FRONTEND=noninteractive -w /esim ubuntu:25.04 \
  bash -c 'ln -fs /usr/share/zoneinfo/Etc/UTC /etc/localtime && apt-get update -qq \
    && apt-get install -y -qq tzdata sudo >/dev/null \
    && ESIM_NO_FANCY=1 bash install-eSim.sh --install; echo EXIT=$?'

# apply patches/fix-canberra-25.04.patch to nghdl/install-nghdl.sh,
# re-run the same command: completes 11/11, ~18 min
```

`ESIM_NO_FANCY=1` is the installer's own kill switch for the fancy UI and gives
plain `>>>` output. No `-t` on purpose: the absence of a pseudo-terminal is what
exposes Issue 2.

The archive query in Issue 1 needs no eSim checkout at all:

```bash
docker run --rm ubuntu:25.04 bash -c \
  'apt-get update -qq && apt-cache policy libcanberra-gtk-module libcanberra-gtk3-module'
```

## Limitations

Stated so the scope of the claims is clear.

- **No VirtualBox run.** Containers only. The apt failure is archive-level, shown
  by the `apt-cache policy` query above, and will reproduce on bare metal. That
  is reasoning from the archive, not a bare-metal test.
- **Whether anything needs the GTK2 canberra module at runtime was not checked.**
  The fix assumes gtk3-only is sufficient because 26.04 already uses it.
- **Whether Issue 2 is already reported.** Searched the issue tracker and open
  pull requests, found nothing on unattended installs or a `--yes` flag. A search
  is not proof of absence.
- **The GUI was never launched.** The installer finishes and the toolchain doctor
  passes. `PyQt6.QtWidgets`, `PyQt6.QtSvg`, `PyQt6.uic` and `PyQt6.Qsci` all
  import on 25.04, so the installer's Qt package set is correct for the release.
  eSim's interface was not started. Installs correctly is not the same as runs
  correctly.
- **eSim's own pytest suite was not exercised.** The suite needs eSim's
  virtualenv on `sys.path` plus `pyqt6-dev-tools` for `PyQt6.uic`. With both in
  place the imports resolve, but the run did not complete within the time
  available and was stopped. The attempts are kept in `logs/` to show the method.
  No conclusion is drawn from them. The suite covers the GUI, not the installer.

## Branch

The task document directs submissions to the `installers` branch, so the files
are committed there. Three facts about that branch are worth stating, because
GitHub reports this fork as far ahead of it.

Upstream `installers` is a separate packaging tree, not a variant of master. It
holds per-OS folders (`ArchLinux`, `Docker`, `Fedora`, `MacOS`, `Ubuntu`,
`Windows`) and per-release Ubuntu scripts under `Ubuntu/install-eSim-scripts/`,
including `install-eSim-25.04.sh`.

That tree is not what eSim 2.6 ships. The published `eSim-2.6.zip` carries the
unified `install-eSim.sh` with `detect_profile()`, and its `install-nghdl.sh` is
byte-identical to master at `d86c189`. No Ubuntu script on upstream `installers`
references `libcanberra` at all, so neither affected file nor the defect exists
there. `MAINTAINERS-PACKAGING.md` in master likewise states there is no separate
installers branch any more.

The submission branch was therefore created from `master`, which is where both
affected files live and where both patches apply. The large ahead/behind count
against upstream `installers` reflects that choice, not unrelated work.

## Files

Both fixes are applied to the source files on this branch:

- `nghdl/install-nghdl.sh` — Issue 1, two lines in `detect_profile()`
- `Ubuntu/install-eSim.sh` — Issue 2, `--yes` flag and the `ask()` helper

The same changes are also provided as standalone diffs, applicable with
`patch -p1` from the repository root:

```
winter-internship-2026-task4/
├── README.md        this report
├── patches/         both fixes as unified diffs
└── logs/            every run referenced above
```
