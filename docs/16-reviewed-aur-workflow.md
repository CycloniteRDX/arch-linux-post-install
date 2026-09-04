# 16 — Add a reviewed AUR workflow with Paru

## Goal

Extend the stable workstation with a controlled way to build and maintain
software that is not available in the enabled official Arch repositories.

This chapter:

- installs the official `base-devel` metapackage;
- performs one manual AUR review and build cycle;
- builds and installs `paru` from its AUR recipe;
- establishes a review-first workflow for future AUR packages;
- keeps official repository upgrades separate from AUR rebuilds;
- records how to inspect, update, rebuild, remove, and recover foreign packages.

The AUR itself is not installed or enabled as a binary repository. It provides
user-contributed package build metadata. `paru` is a convenience client around
that workflow; it does not turn an AUR package into an official Arch package or
make its recipe trustworthy.

This chapter does not add an unofficial binary repository, enable automatic
upgrades, grant passwordless privilege, run a package build as root, or make an
AUR package part of the minimum workstation baseline.

## Prerequisites

- Chapters 00 through 15 are complete.
- The workstation passed the chapter 14 readiness gate.
- The first stable project baseline is available as tag `v1.0.0`.
- The four project repositories are clean and published.
- NetworkManager provides working internet access and DNS.
- The current system has a recent verified backup.
- The verified Arch installation USB and LUKS passphrase remain available.
- There is enough time and free space to compile software and diagnose a failed
  package build.

Run the chapter as the regular user `neon`. Use `sudo` only where the command
shows it. A `PKGBUILD` executes as the user running `makepkg`; it can therefore
read or modify that user's accessible files even though it cannot silently
become root.

## Fix the trust boundary before installing anything

The relevant objects have different owners:

| Object | Owner and trust boundary |
| --- | --- |
| Official repository package | Built and signed through the Arch packaging infrastructure. |
| AUR Git repository | User-contributed recipe and related files; review is the local user's responsibility. |
| Upstream source | Code or binary fetched by the recipe; its identity and integrity must be checked separately. |
| Local package archive | Result produced on this workstation by `makepkg`. |
| Installed foreign package | Tracked by pacman, but not updated by official repository metadata. |
| `paru` | Optional helper that automates parts of the same AUR process. |

Neither HTTPS, AUR votes, comments, a checksum, nor an AUR helper covers the
whole chain:

- HTTPS authenticates and protects one transport connection;
- a checksum proves that downloaded bytes match the value written in the
  recipe, not that either is benign;
- a valid upstream signature authenticates a source according to its signing
  key, not the surrounding `PKGBUILD`;
- votes and popularity show use, not review quality;
- pacman tracks the final archive but does not certify its original recipe;
- `paru` can show changes but cannot decide whether they are safe.

The selected policy is therefore **review, build as an unprivileged user, then
let pacman install the resulting archive**.

## Record the clean starting state

Read current Arch news and perform the ordinary maintenance checks before
introducing a second package source:

```bash
checkupdates
pacman -Qm
pacman -Qdt
sudo pacdiff --output
systemctl --failed --no-pager
systemctl --user --failed --no-pager
df -hT / /home
```

Record the complete `pacman -Qm` output in the private system record. An empty
result is expected for the validated baseline unless a deliberately retained
foreign package has already been documented. Do not remove an unexpected item
until its origin and purpose are understood.

Create a current Restic snapshot before the package-toolchain transaction if
important work has changed since the last verified backup.

## Install the official package-building toolchain

Perform one complete official-repository transaction:

```bash
sudo pacman -Syu --needed base-devel
```

`base-devel` is an official metapackage whose dependencies provide the standard
Arch package-building toolchain, including `make`, `gcc`, `binutils`,
`pkgconf`, `fakeroot`, and related utilities. Keeping the metapackage installed
also records that the complete toolchain is intentionally required; its
dependencies should not later appear as disposable build leftovers.

Verify the metapackage and the tools used below:

```bash
pacman -Qi base-devel
pacman -Q git base-devel fakeroot
command -v git makepkg fakeroot gcc make
makepkg --version
```

Resolve any `.pacnew` or hook result before continuing:

```bash
sudo pacdiff --output
systemctl --failed --no-pager
```

If the transaction changed the kernel, microcode, mkinitcpio, systemd,
systemd-boot, or sbctl, verify the boot chain before any reboot:

```bash
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

Do not use `pacman -Sy base-devel`; the `-u` is required to preserve Arch's
complete-upgrade model.

## Create a persistent review workspace

Keep manually reviewed AUR Git repositories outside `Downloads`, `/tmp`, the
project repositories, and the dotfiles tree:

```bash
mkdir -p ~/Builds/aur
chmod 0700 ~/Builds ~/Builds/aur
ls -ld ~/Builds ~/Builds/aur
```

This directory contains public recipes and build products, not secrets. The
restrictive permissions prevent another local user from casually modifying a
recipe between review and build. It is not a sandbox and does not make the
recipe safe.

## Obtain the Paru recipe without executing it

Stop if a path named `~/Builds/aur/paru` already exists. Inspect and reconcile
that directory instead of cloning over it.

```bash
test ! -e ~/Builds/aur/paru
cd ~/Builds/aur
git clone https://aur.archlinux.org/paru.git
cd paru
git status --short --branch
git remote get-url origin
git log --oneline --decorate -n 10
```

The remote must be exactly the HTTPS AUR Git URL intentionally requested. The
working tree must be clean. Git commit history provides change history; AUR
recipe commits are not automatically a cryptographic endorsement by Arch or by
the upstream Paru author.

List the small recipe repository before running any package command:

```bash
find . -maxdepth 1 -type f -printf '%f\n' | sort
git status --short
```

Do not run `makepkg`, `paru`, or any script from the clone until the next review
is complete.

## Perform the first manual recipe review

Open every package-control file and patch in the repository:

```bash
bat PKGBUILD .SRCINFO
find . -maxdepth 1 -type f \( -name '*.install' -o -name '*.patch' -o -name '*.diff' \) -print
```

If the second command finds additional files, inspect each exact path with
`bat`. Then review the complete `PKGBUILD`, including shell code outside the
named packaging functions.

Confirm at least:

1. `pkgname`, `pkgver`, `pkgrel`, architecture, licence, and upstream URL name
   the intended project.
2. Runtime, build, and check dependencies have plausible purposes.
3. Every `source=()` entry points to the expected upstream or to a reviewed
   package-repository file.
4. Release sources are pinned to the intended version rather than fetching an
   unrelated moving target.
5. Checksums and any `validpgpkeys` entries match the verification model.
6. `prepare()`, `build()`, `check()`, and `package()` contain commands expected
   for this project.
7. No command requests `sudo`, changes live system files, starts services, reads
   private user data, or downloads and executes an unreviewed second script.
8. Any `.install` script, patch, or generated desktop/service file is understood.

Searches can help locate important fields but never replace reading the file:

```bash
rg -n '^(pkgname|pkgver|pkgrel|arch|url|license|depends|makedepends|checkdepends|source|sha256sums|validpgpkeys)' PKGBUILD
rg -n '^[[:space:]]*(prepare|pkgver|build|check|package)[[:space:]]*\(\)' PKGBUILD
bash -n PKGBUILD
```

`bash -n` checks shell syntax only. It does not assess intent, source safety,
dependencies, filesystem effects, or commands assembled dynamically.

Only after the human-readable recipe has been reviewed, compare its generated
metadata with the committed `.SRCINFO`:

```bash
diff -u .SRCINFO <(makepkg --printsrcinfo)
```

No output means the generated and committed metadata agree. A difference must
be explained before building. Agreement still does not prove that the recipe
is safe because `.SRCINFO` is only a metadata projection of the `PKGBUILD`.

## Download and verify sources before compiling

After the recipe review succeeds:

```bash
makepkg --verifysource
```

This downloads the declared sources and checks the recipe's hashes and PGP
requirements without compiling the package. Read the complete output.

If an unknown PGP key is required:

1. copy its full fingerprint;
2. verify that fingerprint from an independent official upstream source;
3. import only that verified fingerprint;
4. repeat `makepkg --verifysource`.

After independently verifying it, import and display the full fingerprint with:

```bash
gpg --recv-keys FULL_FINGERPRINT
gpg --fingerprint FULL_FINGERPRINT
```

Do not copy a short key ID from an AUR comment and do not replace the failed
check with `--skippgpcheck`, `--skipchecksums`, or edited `SKIP` entries.

For VCS sources, a `SKIP` checksum can be technically intentional because the
checkout changes; it increases the importance of verifying the repository URL,
the selected tag or commit, and any upstream signature policy.

## Build and install Paru as the regular user

From `~/Builds/aur/paru`, run exactly:

```bash
makepkg -Ccsri
```

There is deliberately no `sudo` before `makepkg`:

- `-C` removes an existing source tree before starting the new build;
- `-c` removes temporary build work after success;
- `-s` asks pacman, through sudo, to install missing declared dependencies;
- `-r` removes dependencies installed only for this build when they are no
  longer required after installation;
- `-i` asks pacman, through sudo, to install the resulting local package.

Read every dependency transaction and the final local-package installation.
The recipe and compiler run as `neon`; only pacman's declared package
transactions cross the privilege boundary.

The two cleanup options are case-sensitive and have different moments of
effect: `-C` cleans the old source tree before the build, while `-c` cleans
`src/` and `pkg/` after a successful build. A completed `makepkg -Csri` run is
still valid, but it intentionally lacks the post-build cleanup and therefore
may leave those two directories behind.

Stop on a failed check or build. Do not add `--nocheck`, edit out a dependency,
run the build as root, or switch to a prebuilt package simply to turn the error
into a successful exit status.

## Verify the installed helper and package ownership

```bash
paru --version
pacman -Qi paru
pacman -Qo "$(command -v paru)"
pacman -Qm
pacman -Qdt
paru -P --stats
```

The `paru` executable must be owned by the installed `paru` package. `paru`
must now appear in `pacman -Qm` because no enabled official repository owns
that package. Record the complete foreign-package inventory again.

`pacman -Qdt` should not gain unexplained orphan candidates. `base-devel`
remains intentionally installed as an explicit metapackage; build-only
dependencies introduced solely for Paru should have been removed by `-r` when
no longer required.

Inspect the effective configuration sources without creating a user override:

```bash
paru --help
man paru
man paru.conf
grep -Ev '^[[:space:]]*(#|$)' /etc/paru.conf
test ! -e ~/.config/paru/paru.conf
```

This baseline keeps the packaged configuration and uses explicit command-line
options for sensitive behavior. Do not enable:

- `SkipReview`, which suppresses the recipe review;
- `NoCheck`, which disables declared package tests;
- `CombinedUpgrade`, which can refresh databases before the review completes;
- `SudoLoop`, which keeps sudo credentials alive during a long build;
- `UseAsk`, which can automate conflict confirmation;
- `CleanAfter`, which discards useful local review evidence automatically.

No Paru daemon, systemd unit, timer, root service, or sudoers rule is needed.
Run Paru as `neon`, never as `sudo paru`; Paru invokes sudo only for the pacman
transactions that actually require privilege and keeps its review state in the
regular user's XDG directories.

## Evaluate a future package before selecting it

Prefer an official package whenever it meets the requirement. Search there
first:

```bash
pacman -Ss 'SEARCH_TERM'
```

Only when no suitable official package exists, search the AUR explicitly:

```bash
paru --aur -Ss 'SEARCH_TERM'
paru --aur -Si PACKAGE
paru -Gc PACKAGE
paru -Gp PACKAGE | bat --language=bash --paging=always
```

Use the exact package name only after checking:

- whether an official package already provides the same program;
- the upstream project's real URL and release state;
- the AUR maintainer, last update, orphaned or out-of-date flags;
- votes, popularity, and comments as context rather than proof of safety;
- dependencies, optional features, patches, and installation hooks;
- whether the package is a regular release, a vendor binary, or a moving VCS
  build.

Prefer a normal release recipe when available. A `-bin` package saves compile
time by installing a prebuilt upstream artifact, while a `-git` package follows
development commits and can rebuild frequently or break unexpectedly. Neither
suffix is inherently malicious, but each changes what must be reviewed and
maintained.

Never paste a command from an AUR comment without independently understanding
why it is needed. Comments can reveal current build problems, but any user can
post incorrect or unsafe instructions.

## Review a new package in a local clone

For the first installation of an important package, download its recipe into
the persistent review area rather than approving an unseen interactive result:

```bash
cd ~/Builds/aur
paru -G PACKAGE
cd PACKAGE
git status --short --branch
git remote get-url origin
git log --oneline --decorate -n 10
find . -maxdepth 1 -type f -printf '%f\n' | sort
bat PKGBUILD .SRCINFO
```

Stop if the target directory already existed unexpectedly. Apply the same
recipe, metadata, patch, source, checksum, signature, and hook review used for
Paru. Then verify sources:

```bash
diff -u .SRCINFO <(makepkg --printsrcinfo)
makepkg --verifysource
```

After the review succeeds, Paru can build and install the already inspected
local recipe:

```bash
paru -Bi .
```

Alternatively, for a previously understood package, request its exact AUR name
while forcing the review and build-dependency prompt:

```bash
paru --aur -S --review --removemake=ask PACKAGE
```

Do not use bare `paru SEARCH_TERM` as the canonical installation method. Its
interactive search is convenient, but an exact reviewed package name makes the
selected source and target clearer.

After every new installation:

```bash
pacman -Qi PACKAGE
pacman -Ql PACKAGE
pacman -Qm
pacman -Qdt
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

Inspect any new system or user unit before enabling it. Installing a package
does not authorize a daemon, autostart entry, kernel module, firewall opening,
sudoers fragment, or unreviewed dotfile.

## Use a two-phase maintenance cycle

The upstream shortcut `paru` means `paru -Syu`, but this project does not use
the bare shortcut. Keeping the two sources visible makes failures and trust
decisions easier to diagnose.

First inspect both update sets without changing the system:

```bash
checkupdates
paru -Qua
pacman -Qm
```

Then use this sequence:

1. Read new Arch Linux news.
2. Ensure a current backup and enough recovery time are available.
3. Upgrade every official repository package:

   ```bash
   sudo pacman -Syu
   ```

4. Resolve package configuration and verify the official transaction:

   ```bash
   sudo pacdiff --output
   systemctl --failed --no-pager
   systemctl --user --failed --no-pager
   ```

5. Review and rebuild AUR packages separately:

   ```bash
   paru -Sua --review --upgrademenu --removemake=ask
   ```

6. Recheck foreign packages, orphans, configuration, and services:

   ```bash
   pacman -Qm
   pacman -Qdt
   sudo pacdiff --output
   systemctl --failed --no-pager
   systemctl --user --failed --no-pager
   ```

7. When boot-related official or foreign packages changed, verify the UKIs,
   systemd-boot, and signatures before rebooting:

   ```bash
   sudo bootctl --esp-path=/boot list
   sudo sbctl verify
   ```

`paru -Sua` limits the upgrade target set to AUR packages while still allowing
required official dependencies to be resolved from the databases just updated
by `pacman -Syu`. The explicit `--review` prevents a later configuration change
from silently making skipped review the intended workflow.

Never skip an official repository package from a complete upgrade. If an AUR
package cannot currently rebuild, preserve the complete official upgrade and
diagnose that foreign package separately.

## Update Paru manually when the helper itself is broken

Paru normally updates itself as part of `paru -Sua`. Its retained manual clone
is the recovery path when that cannot run:

```bash
cd ~/Builds/aur/paru
git status --short --branch
git fetch origin
git log --oneline --decorate HEAD..origin/master
git diff --stat HEAD..origin/master
git diff HEAD..origin/master -- PKGBUILD .SRCINFO '*.install' '*.patch' '*.diff'
```

Review all fetched changes, not only the version number. If the working tree is
clean and the update is acceptable:

```bash
git merge --ff-only origin/master
bat PKGBUILD .SRCINFO
diff -u .SRCINFO <(makepkg --printsrcinfo)
makepkg --verifysource
makepkg -Ccsri
```

The fast-forward-only merge preserves the AUR history and refuses an accidental
local divergence. If local recipe changes exist, stop and decide whether to
retain, rebase, or discard them; do not hide them with a forced reset.

## Rebuild a foreign package after an ABI transition

An AUR version can remain unchanged while an official shared library, compiler,
or language runtime changes. If the installed program then reports a missing
library or incompatible ABI, first complete the official upgrade and identify
the exact package and dependency.

For a known reviewed package, force a fresh target build:

```bash
paru --aur -S --review --rebuild=yes --removemake=ask PACKAGE
```

Then verify the original error and package ownership again. Do not create a
manual compatibility symlink between different library versions and do not
downgrade an arbitrary official dependency to match one stale foreign package.

## Remove an AUR package through pacman

Once installed, an AUR-built archive is still a pacman-managed package. Inspect
it and its reverse dependencies:

```bash
pacman -Qi PACKAGE
pactree -r PACKAGE
```

Remove the package and dependencies that are no longer required:

```bash
sudo pacman -Rs PACKAGE
```

Read the complete removal list before confirming. Then inspect rather than
blindly delete remaining candidates:

```bash
pacman -Qm
pacman -Qdt
sudo pacdiff --output
```

Removing a package does not automatically delete its user data, cache, or
manually created configuration. Identify those paths separately and move them
to Trash only when their contents are no longer needed.

## Inspect build and helper storage without automatic deletion

```bash
du -sh ~/Builds/aur ~/.cache/paru 2>/dev/null
find ~/Builds/aur -maxdepth 3 -type f -name '*.pkg.tar.*' -printf '%p\n'
find ~/.cache/paru -maxdepth 4 -type f -name '*.pkg.tar.*' -printf '%p\n' 2>/dev/null
```

Retaining a recent built archive and the reviewed Git recipe can help diagnose
or temporarily recover one package. It is not a system snapshot and may not be
compatible after other packages change.

Keep `~/Builds/aur/paru` as the reviewed recipe and manual recovery clone. It
is not a duplicate installation. If an earlier successful `makepkg -Csri` run
left only the temporary `src/` and `pkg/` directories, inspect them first:

```bash
cd ~/Builds/aur/paru
du -sh src pkg 2>/dev/null
find . -maxdepth 1 -type d \( -name src -o -name pkg \) -print
```

They can be moved to Trash after Paru and the installed package have passed the
completion checkpoint:

```bash
cd ~/Builds/aur/paru
gio trash ./src ./pkg
gio trash --list
```

This cleanup removes only reproducible build work. Retain the Git repository,
`PKGBUILD`, `.SRCINFO`, downloaded sources, and recent package archive unless a
later deliberate storage review decides otherwise. Future builds use
`makepkg -Ccsri`, which performs both the pre-build and post-build cleanup.

Do not make `paru -Sc` or `paru -Scc` part of routine maintenance. Paru extends
cache cleanup to its AUR data, and aggressive cleanup can remove the exact
source or archive needed to understand a failed update. The existing
`paccache.timer` remains responsible for conservative official package-cache
retention.

## Failure and recovery cases

### The recipe review finds an unexplained command

Stop before `makepkg`. Compare the AUR history, upstream build instructions,
package comments, and previous recipe version. Do not assume the maintainer's
intent or rely on popularity.

### Source verification fails

Treat a checksum or PGP failure as a stop condition. Determine whether the
upstream source was legitimately replaced, the recipe is stale, the required
key is missing, or the fetched content is wrong. Never bypass the check as a
generic fix.

### The build or test fails

Keep the full terminal output and inspect the first meaningful failure. Confirm
that the official system is fully upgraded and retry the underlying `makepkg`
workflow before blaming Paru. Do not use `--nocheck` merely because a declared
test fails.

### Pacman reports a database lock

Check for a real pacman, makepkg, or Paru process before touching the lock:

```bash
ps -ef | rg '[p]acman|[m]akepkg|[p]aru'
sudo lsof /var/lib/pacman/db.lck 2>/dev/null
```

Never run Pacman and Paru concurrently. Remove a stale lock only after proving
that no package transaction or hook remains and examining the interrupted
transaction.

### Paru stops after the official upgrade

The official upgrade remains valid. Inspect `sudo pacdiff --output`, failed
units, the Paru error, the recipe changes, and the cached clone. Do not roll
back the complete Arch system merely because one optional AUR package failed to
rebuild.

### An AUR entry becomes orphaned, out of date, or disappears

The installed package remains visible in `pacman -Qm`. Decide whether to keep it
temporarily, adopt or maintain a recipe, replace it, or remove it. An installed
package does not become trusted or maintained merely because it still runs.

### Paru itself cannot run

Pacman remains the authority for official upgrades and all installed package
records:

```bash
sudo pacman -Syu
pacman -Qi paru
pacman -Qm
```

Repair Paru from its manually retained clone or remove it with pacman. Never
make an AUR helper necessary for boot, networking, package-database recovery, or
the official upgrade path.

## Rollback

Remove only the helper while retaining the ability to build manually:

```bash
sudo pacman -Rs paru
```

If the workstation will no longer build any Arch or AUR package, inspect the
transaction offered for the metapackage separately:

```bash
sudo pacman -Rs base-devel
```

Do not confirm if the removal list contains a compiler or tool still needed for
university, development, kernel modules, or another documented workflow.

The retained recipe and Paru cache are user data. Move them to Trash rather
than permanently deleting them when rollback is intentional:

```bash
gio trash ~/Builds/aur/paru
gio trash ~/.cache/paru
gio trash --list
```

Finally:

```bash
pacman -Qm
pacman -Qdt
sudo pacdiff --output
```

## Completion checkpoint

```bash
pacman -Q base-devel paru
command -v git makepkg fakeroot gcc make paru
paru --version
pacman -Qo "$(command -v paru)"
pacman -Qm
pacman -Qdt
paru -Qua
sudo pacdiff --output
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

The chapter is complete when:

- `base-devel` is installed from the official `core` repository;
- the Paru AUR Git repository was read before any recipe command executed;
- `.SRCINFO`, sources, checksums, signature requirements, functions, patches,
  and hooks were reviewed;
- `makepkg` ran as `neon`, never as root;
- the resulting `paru` executable is owned by the foreign `paru` package;
- the reviewed `~/Builds/aur/paru` clone remains available as the manual
  recovery path;
- the complete `pacman -Qm` inventory is understood and recorded;
- no unexplained orphan remains after the build;
- official upgrades and AUR upgrades use the separate documented phases;
- no automated upgrade, unofficial binary repository, daemon, timer, sudoers
  rule, `SkipReview`, or generic verification bypass was introduced;
- Pacman remains sufficient for official upgrades and removal if Paru fails.

This complete sequence, including the later AUR update check, passed hardware
validation on the target ThinkPad T14 Gen 1 AMD on 2026-09-04. The validated
first build used `makepkg -Csri`; the absence of lowercase `-c` affected only
post-build cleanup and did not affect the resulting package or installation.

## Sources

- [ArchWiki: Arch User Repository](https://wiki.archlinux.org/title/Arch_User_Repository)
- [ArchWiki: Makepkg](https://wiki.archlinux.org/title/Makepkg)
- [ArchWiki: Creating packages](https://wiki.archlinux.org/title/Creating_packages)
- [Arch package: base-devel](https://archlinux.org/packages/core/any/base-devel/)
- [Arch manual: makepkg(8)](https://man.archlinux.org/man/makepkg.8)
- [Arch manual: PKGBUILD(5)](https://man.archlinux.org/man/PKGBUILD.5)
- [Arch manual: pacman(8)](https://man.archlinux.org/man/pacman.8)
- [Paru upstream README](https://github.com/Morganamilo/paru)
- [Paru upstream manual](https://github.com/Morganamilo/paru/blob/master/man/paru.8)
- [Paru configuration manual](https://github.com/Morganamilo/paru/blob/master/man/paru.conf.5)

## Next step

The next independent extension is Qt 6 appearance integration without changing
the GTK or modular desktop ownership model.
