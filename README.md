# Arch Linux Post-install

An ordered procedure for turning the first working Arch Linux TTY into a
secure, maintainable, and complete Niri-based daily-driver workstation.

> [!NOTE]
> This repository starts after `arch-linux-runbook`. It is not an alternative
> installer and must not be followed on an unidentified or partially upgraded
> system.

## Project status

Foundation draft. The target profile and complete roadmap are defined, but the
implementation chapters still need to be written, reviewed, and tested on the
target ThinkPads.

## Starting point

This procedure assumes the companion runbook has produced a machine with:

- A bootable LUKS2-on-LVM Arch Linux installation.
- Separate ext4 filesystems for `/` and `/home`.
- A FAT32 EFI System Partition mounted at `/boot` with root-only masks.
- Signed normal and fallback UKIs, systemd-boot, and working Secure Boot.
- A 16 GiB encrypted disk-swap LV without hibernation.
- A regular user named `neon` with password-protected `sudo` access.
- NetworkManager and systemd-timesyncd enabled.
- A successful TTY login and no required graphical environment.

The exact assumptions are fixed in the
[post-installation profile](docs/00-post-installation-profile.md). Stop and
reconcile any difference before applying later chapters.

## Reference method

The ArchWiki [General recommendations](https://wiki.archlinux.org/title/General_recommendations)
page is used as the main annotated index. It is not treated as a checklist in
which every possible service must be installed.

Each subject is classified as:

- **Canonical:** required by this workstation profile.
- **Conditional:** configured only when the related hardware or use case is
  present, such as printing or network shares.
- **Optional:** useful but not required for completion.
- **Deferred:** explanation, alternatives, or troubleshooting belonging in the
  handbook.

Commands and package names must still be checked against current Arch manuals,
official package metadata, and upstream documentation before publication.

## Scope

This repository owns system-wide integration after the first boot:

- Package-maintenance, mirror, cache, news, and AUR policies.
- Logging, enabled-service review, timers, and routine maintenance.
- Periodic TRIM, encrypted discard propagation, zram, and disk-swap fallback.
- Firewall, exposed-service review, and practical workstation hardening.
- Firmware updates, suspend, battery policy, thermals, and ThinkPad hardware.
- AMD graphics, PipeWire, Bluetooth, printing, removable media, polkit,
  portals, secrets, and XWayland compatibility.
- Fonts, localization, XDG user directories, and console utilities.
- Daily applications and their required system packages.
- Niri, Kitty, greetd, and the packages underlying the selected desktop
  components.
- Backup, recovery, and final daily-driver verification.

## Repository boundaries

| Repository | Responsibility |
| --- | --- |
| `arch-linux-runbook` | Install and verify the encrypted, Secure Boot-capable TTY system. |
| `arch-linux-post-install` | Install packages, configure system files, and enable system services. |
| `niri-dotfiles` | Own reviewed files below the user's configuration directories. |
| `arch-linux-handbook` | Explain concepts, alternatives, diagnostics, and reusable guides. |

This repository may document how a desktop component integrates with the
system, but it must not become a second copy of the user's dotfiles. Secrets,
machine-generated state, tokens, private keys, and Wi-Fi credentials are never
committed.

## Execution strategy

The first stages run from TTY and establish a known-good, fully updated base.
Niri and Kitty are then installed as an early graphical checkpoint. Chapter 05
coordinates with `niri-dotfiles` for only the minimal reviewed configuration
needed to open Kitty and exit the session. The first Niri session is started
manually from TTY; automatic graphical login is not enabled yet.

After that checkpoint, later commands may be run from Kitty. Greetd, lock and
idle handling, bars, launchers, notifications, wallpaper, and themes are added
only after the manual Niri session is reliable. A broken desktop must never be
the only available administrative path.

## Design rules

- Use complete system upgrades; never document partial upgrades.
- Prefer official repositories and upstream-supported components.
- Install packages in small, reviewable groups with stated responsibilities.
- Distinguish canonical, conditional, and optional components.
- Verify every modified file, enabled service, timer, and boot artifact.
- Avoid overlapping network, time, power, audio, or firewall daemons.
- Keep hibernation outside the canonical profile.
- Do not enable `sshd.service` merely because the OpenSSH client is installed.
- Start Niri manually before installing or enabling a greeter.
- Keep user configuration in `niri-dotfiles` and theory in the handbook.
- End every chapter at a recoverable checkpoint.

## Build sequence

See the [post-install roadmap](docs/README.md).

## Related repositories

- [Arch Linux Runbook](https://github.com/CycloniteRDX/arch-linux-runbook)
  defines the starting system.
- [Arch Linux Handbook](https://github.com/CycloniteRDX/arch-linux-handbook)
  explains concepts and trade-offs.
- [Niri Dotfiles](https://github.com/CycloniteRDX/niri-dotfiles) deploys the
  reviewed user configuration.

## Source policy

The repository follows current Arch Linux and upstream documentation. The
offline ArchWiki snapshot is useful for research, but version-sensitive commands
must be rechecked before they are committed or executed.
