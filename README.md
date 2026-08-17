# Arch Linux Post-install

An ordered procedure for turning the first working Arch Linux TTY into a
secure, maintainable, and complete Niri-based daily-driver workstation.

## Project status

Early design stage. Components and package choices will be selected only after
their responsibilities and integration requirements have been reviewed.

## Starting point

This procedure assumes the companion runbook has produced a machine with:

- A bootable encrypted Arch Linux installation.
- A signed unified kernel image and working Secure Boot.
- A regular user with working `sudo` access.
- NetworkManager enabled.
- A successful TTY login.

## Scope

This repository owns system-level configuration after the first boot:

- Package maintenance and mirror policy.
- Clock synchronization and routine timers.
- Zram, disk swap policy, TRIM, and encrypted discard decisions.
- Firewall and workstation security baseline.
- Firmware updates, power management, thermals, and suspend.
- Graphics, audio, Bluetooth, printing, removable media, and portals.
- Fonts, localization, XDG user directories, and secrets integration.
- Daily applications such as a browser, terminal, file manager, PDF reader,
  archive tools, media tools, and development utilities.
- Niri and all system packages required by the selected desktop stack.
- Greetd, session startup, notifications, status UI, launcher, lock screen,
  idle handling, screenshots, wallpaper, theming, and logout integration.
- Final verification and recovery notes.

It does not own detailed conceptual tutorials or the contents of user dotfiles.

## Design rules

- Build the workstation in small, bootable stages.
- Explain package roles before installing large groups.
- Distinguish required components from optional applications.
- Verify every enabled service.
- Avoid overlapping daemons that perform the same job.
- Record decisions before committing to a desktop component.
- Keep hibernation out of the canonical profile unless it becomes a real
  requirement.
- Link to the handbook for theory and troubleshooting.
- Hand user configuration to the dotfiles repository only after the underlying
  packages and services are installed.

## Build sequence

See the [post-install roadmap](docs/README.md).

## Related repositories

- [Arch Linux Runbook](https://github.com/CycloniteRDX/arch-linux-runbook)
  defines the starting system.
- [Arch Linux Handbook](https://github.com/CycloniteRDX/arch-linux-handbook)
  explains concepts and trade-offs.
- [Niri Dotfiles](https://github.com/CycloniteRDX/niri-dotfiles) deploys the
  reviewed user configuration.
