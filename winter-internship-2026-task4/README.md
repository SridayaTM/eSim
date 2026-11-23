# eSim 2.6 on Ubuntu 25.04: installation failure, root cause, fix

Screening Task 4, FOSSEE Winter Internship 2026
Tested 20-23 November 2026.

## Result

eSim 2.6 does not install on Ubuntu 25.04. `install-eSim.sh --install` aborts at
step 7 of 11. The downloads page lists 25.04 as supported.

Cause: a one-release error in the dependency table of `nghdl/install-nghdl.sh`.
Fix: two lines. After patching, the same install completes all 11 steps and the
toolchain doctor reports every component OK.

A second defect turned up during testing: the installer cannot run without a
terminal. Five prompts, no flag to answer them. Patched as well, 87 lines.

Two issues, two fixes, both verified by running them.

## What was tested

| | |
| --- | --- |
| Artifact | `eSim-2.6.zip`, esim.fossee.in/downloads, 86,783,803 bytes |
| SHA256 | `79ac33c315b142308c078f00a4d4383335cadbdaef1d2841231acba3504e5b8c` |
| Matches published hash | yes |
| OS | `ubuntu:25.04` container, clean per run |
| Source refs | `master` @ `d86c189`, tag `v2.6` |

The release ships `install-nghdl.sh` identical to `d86c189` (`diff`, no output),
so line numbers below apply to both the zip and the repo.

### Why 2.6 and not 2.5

The task says eSim 2.5 on Ubuntu 25.04. The downloads page lists 2.5 as
supporting Ubuntu 22.04, 23.04 and 24.04. It does not claim 25.04. A failure
there would be an unsupported configuration, not a bug. 2.6 is the current
download and the first to claim 25.04, so that is the version under test.

### Why containers and not VirtualBox

Every run starts from an identical clean state, which matters when you are
attributing a failure to one cause. The defect is an apt archive-level failure,
not a container artefact (see Issue 1, "Not the container"). GUI runtime was not
tested. See Limitations.

---

# Issue 1: install aborts at step 7 on Ubuntu 25.04

Blocks the whole install. Nothing usable is produced.

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

Steps 1-6 complete. KiCad 8 and the symbol library are installed. Steps 7-11
never run, so there is no ngspice, no SKY130 PDK, no launcher and no toolchain
check. eSim cannot start or simulate.

Log: `logs/run-25.04-PUBLISHED-ZIP.log`

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

Upstream: Launchpad #2111252 records the same apt error on 25.04, with the
Ubuntu maintainer confirming the package is absent from plucky. Removed because
GTK2 is deprecated, Debian #967569. Removed, not renamed. There is no
replacement binary.

25.04 sits one branch too high in the case statement.

### Not the container

`universe` was enabled. Step 5 of the same run installed KiCad 8 from universe
in 1m54s, which is what `detect_profile()` specifies for 25.04
(`KICAD_SOURCE="universe"`). The repository worked. The package is not there.

### Present in the shipped zip

`eSim-2.6.zip` contains `nghdl.zip`, unpacked by the installer at step 7.
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

Same file, same lines. The bug is in what users download.

### Fix

`patches/fix-canberra-25.04.patch`

```
117c117
<         26.04)
>         26.04|25.04)
120c120
<         23.04|24.04|25.04)
>         23.04|24.04)
```

23.04 and 24.04 keep the GTK2 module. It is still published for those releases.
25.04 joins 26.04 on gtk3-only.

Suggested follow-up for the maintainers, not implemented here: check apt for an
installation candidate before requesting the GTK2 package. The two lines fix
25.04. A candidate check stops the next release doing this again.

### Verified

Same container, same command, patched tree:

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

Step 11 is `runToolchainDoctor`. Its output marks `[OK]` for ngspice-45, all
eight code models including `ghdl.cm`, the ivlng adapter, Icarus Verilog 14.0
with `vvp` and `libvvp`, Verilator 5.032 and GHDL 5.0.1.

Logs: `logs/run-25.04-fixed-full.log`,
`logs/eSim-install-PUBLISHED-ZIP-FIXED-success.log`

### Issue #637

Open since 20 August 2026. Item 3 of that report is this apt error. Two
differences.

It gives the cause as the package being "obsolete/renamed under Ubuntu 24.04
repositories". The package is still published for noble. It was removed in
plucky, not renamed.

It was filed against a different tree. Its paths (`scripts/setup-esim.sh`,
`/3rdparty/template/sym-lib-table`, `ghdl-4.1.0.tar.gz`) are not in master, and
the reporter states they were on branch `installer`. `detect_profile()` did not
exist there.

The symptom is known. The location and cause in the current installer are new.

---

# Issue 2: `--install` cannot run unattended

Blocks CI and automated verification. No effect on a user at a keyboard.

### Symptom

`read -rp` is called five times on the install path.

| Line | Prompt | Reached |
| --- | --- | --- |
| 1226 | proxy? (y/n) | every run, `setupProxy` |
| 1228-1231 | proxy host / port / user / password | only if proxy=y (1231 is `read -rsp`) |
| 1079 | install IHP Open PDK? (y/n) | every run, `installIhpPdk` |
| 801 | remove /usr/local GHDL? (y/n) | only if a stray GHDL exists |
| 897 | remove it and install KiCad >= N? (y/n) | only if an old KiCad exists |

`case "$option"`, line 1293, accepted `--dry-run`, `--install`, `--uninstall`.
No `--yes`. No environment override.

With stdin at `/dev/null` the installer dies at step 0 of 11.

### Why this reads as an oversight

Three things in the same codebase assume unattended runs work.

1. Lines 1314-1324 export `NEEDRESTART_MODE=l`, with a comment that needrestart
   "BLOCKS on a keypress, so an unattended install silently stalls, looking
   hung, for as long as nobody is watching".
2. `ui_step()` (375) and `ui_log()` (416) print `>>> [N/TOTAL] <label>` whenever
   `UI_LEVEL < 1`, i.e. any non-TTY run. A machine-readable format, on purpose.
3. `Ubuntu/bootstrap.sh` has `require_tty()`, commented: "install/uninstall
   prompt for proxy, sudo password, IHP, etc. Under `curl | bash` stdin is the
   script pipe, so those reads must come from the terminal instead." Line 265
   runs `--install < /dev/tty`. Line 269 runs `--dry-run < /dev/null`.

So the terminal dependency was known and worked around for humans. `--dry-run`
already runs without a terminal. `--install` did not.

### Four consequences, all reproduced

1. A failing `read` kills the install. At EOF `read` returns non-zero,
   `--install` runs under `set -e` with an ERR trap, installer exits.
2. The error names the wrong thing. `ui_step` prints the label before calling
   the function, so a dead prompt in `installIhpPdk` reported "Failed at step
   9/11 Installing the IHP Open PDK". No PDK install was attempted.
3. Piping answers does not work. `printf 'n\nn\n' |` failed at step 9: the
   5-minute NGHDL build spawns children that inherit and consume the same stdin.
   The second answer was gone before it was needed.
4. debconf preconfiguration is skipped silently. Non-TTY runs log
   `dpkg-preconfigure: unable to re-open stdin` after falling through three
   frontends, then continue. Nothing broke here. A package that genuinely needs
   preconfiguration would not get it.

### Fix

`patches/fix-noninteractive-install.patch`, 87 lines, six hunks, against
`Ubuntu/install-eSim.sh`.

- `ask()` helper: variable name, environment override, default, prompt.
  Precedence is env var, then `--yes` default, then prompt as before.
- `--yes` / `-y` as a flag. `ESIM_ASSUME_YES=1` as an environment equivalent.
- The four install-path prompts converted to `ask`.
- Argument guard strips `--yes` before counting. Without this the existing
  `[ "$#" -ne 1 ]` check rejects `--install --yes` before the case runs.
- Usage text updated.

All defaults are `n`. Nothing is deleted, nothing optional is added silently.
Each decision is logged as `non-interactive: <prompt> -> n (default)`.

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
no piped answers, no human. 11/11 and the doctor passes.

Log: `logs/eSim-install-YES-FLAG-success.log`

### Still not CI-ready

`--yes` handles eSim's prompts only. Two things remain, both listed below under
Other observations: `sudo` is called unconditionally, and `DEBIAN_FRONTEND`
does not survive `sudo`. A complete non-interactive mode would resolve `sudo`
once with a root fallback and pass `DEBIAN_FRONTEND` through to apt.

---

## Other observations

Not defects for a desktop install. All three affect automated runs.

**`sudo` called unconditionally.** Root environment with no `sudo` binary: fails
at line 833, `sudo: command not found`, exit 127. Usual pattern is to resolve
sudo once and fall back to empty when already root.

**`DEBIAN_FRONTEND` does not survive `sudo`.** sudoers defaults to `env_reset`,
so setting it before running the installer has no effect on its `sudo apt-get`
calls. Observed as tzdata's geographic-area menu blocking a container run.

**`test_install.sh` step counts disagree.** Header says twelve install steps.
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

# apply patches/fix-canberra-25.04.patch to nghdl/install-nghdl.sh
# re-run the same command: completes 11/11, ~18 min
```

`ESIM_NO_FANCY=1` is the installer's own kill switch for the fancy UI and gives
plain `>>>` output. No `-t` on purpose: the absence of a pseudo-terminal is what
exposes Issue 2.

## Limitations

- GUI never launched. The installer finishes and the doctor passes. eSim's
  interface was not started. Installs correctly is not the same as runs
  correctly.
- No VirtualBox run. Containers only. The apt failure is archive-level so it
  will reproduce on bare metal, but that is reasoning, not a test.
- Whether anything needs the GTK2 canberra module at runtime was not checked.
  The fix assumes gtk3-only suffices because 26.04 already uses it.
- Whether Issue 2 is already reported. Searched the issue tracker and open pull
  requests, found nothing on unattended installs or a `--yes` flag. A search is
  not proof.

## Questions

1. Is the GTK2 canberra module needed at all, or can it go for every release
   rather than per-release?
2. Is a `--yes` mode wanted upstream? It is the prerequisite for any CI job that
   runs the real installer, and `Ubuntu/tests/test_install.sh` currently stubs
   all install steps because the real ones cannot run unattended.
3. The task document says to commit to the `installers` branch.
   `MAINTAINERS-PACKAGING.md` says there is no separate installers branch any
   more. Files are here as instructed. Both patches apply to `master`, where
   the affected files live.
