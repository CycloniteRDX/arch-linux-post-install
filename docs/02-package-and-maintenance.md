# 02 — Establish package and maintenance policy

## Goal

Perform the first complete system upgrade, install the small set of maintenance
and repository tools needed by later chapters, and establish a repeatable Arch
Linux update workflow.

This chapter makes the following changes:

- upgrades every package from the enabled official repositories;
- installs `git`, `openssh`, and `pacman-contrib`;
- enables the weekly `paccache.timer`;
- optionally replaces the mirror list only if the current one is demonstrably
  unsuitable;
- reboots once and verifies the signed boot chain again.

It does not enable automatic system upgrades, `sshd.service`, Reflector's
timer, an AUR helper, or any testing repository.

## Prerequisites

- Chapter 01 completed without unexplained differences.
- NetworkManager provides working internet access and DNS.
- The clock is synchronized.
- The verified Arch installation USB and LUKS passphrase are available.
- There is enough time to diagnose an unexpected update problem before the
  computer is needed for important work.

Run the commands as `neon`. Read the complete transaction summary and every
post-transaction message instead of confirming prompts automatically.

## Update policy

Arch is a rolling-release distribution and supports complete system upgrades.
The canonical update command is:

```bash
sudo pacman -Syu
```

Do not use `pacman -Sy` on its own. It refreshes the repository databases
without upgrading the installed packages and can create an unsupported partial
upgrade. Likewise, do not use `--overwrite`, `--nodeps`, `-Rdd`, or
`--noconfirm` to force an unexplained transaction through.

If a `pacman -Syu` transaction fails after synchronizing the databases, resolve
the reported problem and complete the full upgrade before installing or
removing other packages.

## Read Arch news before upgrading

Before every upgrade, visit the official
[Arch Linux news page](https://archlinux.org/news/) and read every entry newer
than the last successful upgrade. News posts describe manual intervention that
cannot be handled safely by package hooks alone.

For this first TTY-only update, use another computer or phone to read the page.
A graphical browser and optional RSS workflow can be added later. Do not
install an unreviewed AUR news hook merely to avoid this check.

Avoid starting a large upgrade immediately before a class, trip, deadline, or
other situation in which the laptop must remain available.

## Inspect the current mirrors

Display the active mirror records inherited from the installation image:

```bash
grep '^Server' /etc/pacman.d/mirrorlist
```

The list must contain at least one uncommented HTTPS server. If downloads work
at a reasonable speed, retain this known-working list for now. A long list is
not itself a fault, and routinely rewriting it before every update adds no
useful reliability.

The optional Reflector procedure later in this chapter is reserved for mirrors
that are unavailable, out of sync, or consistently slow.

## Perform the first full upgrade

Use one transaction to upgrade the complete system and install the tools needed
by this and later chapters:

```bash
sudo pacman -Syu git openssh pacman-contrib
```

The packages have distinct responsibilities:

| Package | Purpose in this project |
| --- | --- |
| `git` | Obtain and inspect the project repositories and later deploy reviewed dotfiles. |
| `openssh` | Provide the SSH client and key tools used by Git remotes. |
| `pacman-contrib` | Provide `checkupdates`, `paccache`, `pacdiff`, and other pacman maintenance tools. |

Installing `openssh` also places the SSH server binary and its systemd unit on
the machine. It does not enable remote login. Do not start or enable
`sshd.service` unless remote access later becomes an explicit requirement with
its own firewall and authentication policy.

During the transaction:

- confirm that packages come from enabled official repositories;
- read dependency and conflict prompts before answering;
- do not suppress signature verification;
- note every `.pacnew` or `.pacsave` message;
- allow kernel, mkinitcpio, sbctl, and systemd-boot hooks to finish;
- stop if any hook reports a fatal error.

## Check for new configuration files

After the transaction, list configuration files that require attention:

```bash
sudo pacdiff --output
```

No output means that `pacdiff` found no tracked `.pacnew`, `.pacsave`, or
`.pacorig` files. If paths are listed, handle each one before continuing:

1. Compare the active file with the new package version.
2. Preserve intentional local settings.
3. Merge new required directives manually.
4. Validate the affected configuration or service.
5. Remove the auxiliary file only after the merge is complete.

Do not blindly replace every active file with its `.pacnew` counterpart.
Conversely, do not leave `.pacnew` files unresolved indefinitely; they can
contain required changes for a newer package version. A detailed merge workflow
belongs in the handbook.

## Verify the updated boot artifacts

Kernel, microcode, mkinitcpio, systemd, and sbctl updates can affect the boot
chain. Verify the generated UKIs and signatures before rebooting:

```bash
sudo bootctl list
sudo sbctl verify
```

Both `arch-linux.efi` and `arch-linux-fallback.efi` must be listed and signed.
The systemd-boot executables must also be signed. As in the runbook, an unsigned
`/boot/vmlinuz-linux` report is expected because no boot entry executes it.

If a saved boot artifact other than `vmlinuz-linux` is unexpectedly unsigned,
do not reboot. First retry the saved signing operations and verify again:

```bash
sudo sbctl sign-all
sudo sbctl verify
```

Stop if a required UKI or systemd-boot executable still fails verification.
Do not disable Secure Boot to hide a broken update path.

## Configure conservative package-cache cleanup

Pacman keeps downloaded packages in `/var/cache/pacman/pkg`. Retaining older
versions makes a local downgrade or reinstall possible, but an unlimited cache
will grow continuously.

Preview what the default cleanup policy would remove:

```bash
sudo paccache --dryrun
```

Enable the supplied weekly timer:

```bash
sudo systemctl enable --now paccache.timer
```

Verify it:

```bash
systemctl is-enabled paccache.timer
systemctl is-active paccache.timer
systemctl list-timers paccache.timer
```

The default service retains the three most recent cached versions of each
package. This project keeps that default and does not create a custom hook after
every transaction.

Do not use `pacman -Scc` for routine maintenance. It empties the cache and
removes the convenient local rollback path. Use more aggressive cleanup only
when disk pressure is real and the consequences are understood.

## Check for available updates safely

`checkupdates` checks pending upgrades with a separate temporary database, so
it does not put the system into a partial-upgrade state:

```bash
checkupdates
```

It prints package names with their installed and available versions. No output
and exit status 2 means that no updates are available; it is not an error.

`checkupdates` is informational. It does not replace reading Arch news or
running a complete `sudo pacman -Syu` transaction.

## Inspect package state without deleting anything

```bash
pacman -Qdt
pacman -Qm
```

`pacman -Qdt` lists orphaned dependencies. `pacman -Qm` lists foreign packages
that are not present in the enabled official repositories, including manually
built AUR packages. A fresh canonical system should normally produce no output
for either command.

Do not pipe the orphan list directly into a removal command. First determine
why each package was installed and whether it is still required.

## Optional: replace a poor mirror list with Reflector

Skip this section when the current mirrors work correctly.

If they are unavailable, out of sync, or consistently slow, install Reflector
through a complete transaction:

```bash
sudo pacman -Syu reflector
```

Back up the current list:

```bash
sudo cp -a /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.before-reflector
```

Generate a geographically reasonable HTTPS list from recently synchronized
mirrors:

```bash
sudo reflector --country Spain,Portugal,France,Germany --age 24 --protocol https --latest 20 --sort rate --save /etc/pacman.d/mirrorlist
```

Inspect the result before using it:

```bash
grep '^Server' /etc/pacman.d/mirrorlist
```

The command must leave multiple active server records. If it fails or produces
an unsuitable list, restore the backup:

```bash
sudo cp -a /etc/pacman.d/mirrorlist.before-reflector /etc/pacman.d/mirrorlist
```

This profile does not enable `reflector.timer`. Mirror changes remain deliberate
maintenance actions rather than an automatic weekly rewrite.

## Verify Git and the SSH client

```bash
git --version
ssh -V
systemctl is-enabled sshd.service
systemctl is-active sshd.service
```

Git and OpenSSH must report their versions. `sshd.service` must remain disabled
and inactive.

Git identity, GitHub email privacy, SSH-key generation, `ssh-agent`, GitHub CLI,
and authenticated pushes are separate user-level subjects. They are not needed
to clone public documentation and will be covered in the handbook instead of
being hidden inside this system-maintenance chapter.

## Obtain the project repositories

On a fresh machine, clone the public repositories over HTTPS so the remaining
documentation and future Niri bootstrap are available locally:

```bash
mkdir -p ~/Projects/CycloniteRDX
cd ~/Projects/CycloniteRDX
git clone https://github.com/CycloniteRDX/arch-linux-runbook.git
git clone https://github.com/CycloniteRDX/arch-linux-post-install.git
git clone https://github.com/CycloniteRDX/arch-linux-handbook.git
git clone https://github.com/CycloniteRDX/niri-dotfiles.git
```

Do not repeat `git clone` for a repository that already exists. Updating an
existing clean clone is a separate `git pull --ff-only` operation performed
inside that repository after reviewing its status.

HTTPS cloning requires no GitHub token for public repositories. Authentication
is configured only when the machine needs to push changes.

## AUR policy

No AUR package is required at this checkpoint. Do not install an AUR helper or
`base-devel` pre-emptively.

If a later selected component exists only in the AUR:

- install `base-devel` from the official repositories;
- obtain that package's build files as the regular user;
- inspect `PKGBUILD`, `.install`, patches, sources, comments, and signatures;
- build as the regular user, never as root;
- install the resulting package through pacman;
- track foreign packages with `pacman -Qm` and update or rebuild them manually.

The AUR is user-contributed and unsupported. Popularity, votes, an AUR helper,
or a valid signature do not replace source review.

## Reboot into the updated system

After the transaction, configuration review, and signature checks succeed:

```bash
sudo systemctl reboot
```

Unlock LUKS and log in as `neon`. Then verify the new boot:

```bash
uname -r
pacman -Q linux git openssh pacman-contrib
systemctl --failed
sudo journalctl -b -p err --no-pager
systemctl is-enabled paccache.timer
sudo sbctl status
sudo sbctl verify
```

Confirm that:

- the updated kernel boots normally;
- the three new packages are installed;
- no system units are failed;
- the current boot journal contains no unexplained errors that began with this
  update;
- `paccache.timer` remains enabled;
- Secure Boot remains enabled;
- both UKIs and systemd-boot remain signed.

## Routine update sequence

Use this order for future maintenance:

1. Ensure recovery media and enough troubleshooting time are available.
2. Read new entries on the Arch Linux news page.
3. Optionally inspect pending updates with `checkupdates`.
4. Run `sudo pacman -Syu` and read the complete output.
5. Run `sudo pacdiff --output` and resolve every listed file.
6. Inspect `systemctl --failed` and the current boot's error-priority journal
   entries; investigate new failures.
7. Verify signed boot artifacts when boot-related packages changed.
8. Restart affected services or reboot, especially after kernel, systemd,
   graphics, firmware, or low-level library updates.

Never automate confirmation of package transactions on this workstation.

## Completion checkpoint

```bash
pacman -Q git openssh pacman-contrib
systemctl is-enabled paccache.timer
systemctl is-enabled sshd.service
sudo pacdiff --output
systemctl --failed
```

The chapter is complete when:

- the full upgrade and all package hooks completed successfully;
- Git, OpenSSH, and pacman-contrib are installed;
- unresolved package messages have been handled;
- `pacdiff --output` lists no pending configuration files;
- `paccache.timer` is enabled with its default three-version policy;
- `sshd.service` is disabled;
- no automatic package-upgrade or Reflector timer was enabled;
- the public project repositories are available locally;
- the updated signed system boots with no failed units.

## Sources

- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [ArchWiki: Pacman](https://wiki.archlinux.org/title/Pacman)
- [ArchWiki: Mirrors](https://wiki.archlinux.org/title/Mirrors)
- [ArchWiki: Reflector](https://wiki.archlinux.org/title/Reflector)
- [ArchWiki: Arch User Repository](https://wiki.archlinux.org/title/Arch_User_Repository)
- [Arch Linux news](https://archlinux.org/news/)
- [Arch manual: pacman(8)](https://man.archlinux.org/man/pacman.8)
- [Arch manual: checkupdates(8)](https://man.archlinux.org/man/checkupdates.8)
- [Arch manual: paccache(8)](https://man.archlinux.org/man/paccache.8)
- [Arch manual: pacdiff(8)](https://man.archlinux.org/man/pacdiff.8)
- [Arch manual: reflector(1)](https://man.archlinux.org/man/reflector.1)

## Next step

Continue with chapter 03 to enable dm-crypt discard propagation, verify
end-to-end TRIM, enable `fstrim.timer`, add zram, and retain encrypted disk
swap as the lower-priority fallback.
