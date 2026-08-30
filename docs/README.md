# Post-install roadmap

The sequence follows the major areas in ArchWiki General recommendations, but
reorders them around safe and useful checkpoints for this ThinkPad profile.
Every implementation chapter must leave the TTY recovery path operational.

Only chapters that already exist are linked.

| Chapter | Primary environment | Outcome |
| --- | --- | --- |
| [00. Post-installation profile](00-post-installation-profile.md) | Reference | Fix the starting state, canonical decisions, exclusions, and repository boundaries. |
| [01. Handoff audit](01-handoff-audit.md) | TTY | Confirm the runbook result, boot path, Secure Boot, mounts, swap, network, time, free space, and recovery access. |
| [02. Package and maintenance baseline](02-package-and-maintenance.md) | TTY | Perform a complete upgrade; establish news, mirrors, cache, logs, Git and SSH client usage, documentation, and AUR policies. |
| [03. Storage and memory](03-storage-and-memory.md) | TTY | Configure LUKS discard propagation, periodic TRIM, zram, disk-swap fallback, and post-change boot verification. |
| [04. Security and network baseline](04-security-and-network.md) | TTY | Configure firewalld, inspect listening services, preserve local recovery, and document the initial hardening boundary. |
| [05. Minimal graphical bootstrap](05-minimal-graphical-bootstrap.md) | TTY to Niri | Install AMD graphics, Niri, Kitty, portals, polkit, PipeWire, and XWayland compatibility; deploy only the minimal reviewed Niri bootstrap from `niri-dotfiles`; start Niri manually. |
| [06. ThinkPad hardware and power](06-thinkpad-hardware-and-power.md) | TTY or Kitty | Configure firmware updates, TLP, battery thresholds, suspend and lid behavior; verify thermals, function keys, touchpad, and TrackPoint without aggressive hardware tuning. |
| [07. Core workstation services](07-core-workstation-services.md) | Kitty | Configure audio, Bluetooth, removable media, secrets, optional printing, and desktop service integration. |
| [08. User environment](08-user-environment.md) | Kitty | Create XDG directories and install fonts, shell and console tools, archive support, clipboard tools, and common utilities. |
| 09. Daily applications | Niri and Kitty | Install and verify the browser, file manager, PDF reader, text editor, media applications, image tools, and development utilities. |
| 10. Desktop components | Niri | Select and integrate the launcher, status UI, notifications, screenshots, wallpaper, themes, and logout interface. |
| 11. Login, lock, and idle lifecycle | TTY and Niri | Add locking and idle handling, test suspend interactions, then enable greetd only after manual sessions are reliable. |
| 12. Backup and recovery | TTY or Kitty | Back up Secure Boot and LUKS recovery material, define user-data backup policy, and rehearse ISO/chroot recovery. |
| 13. Full dotfiles handoff | Niri | Replace the chapter 05 bootstrap with the complete reviewed `niri-dotfiles` configuration without copying secrets or generated machine state. |
| 14. Daily-driver verification | TTY and Niri | Test boot, updates, shutdown, suspend, networking, audio, Bluetooth, media, external displays, portals, locking, recovery, and backups. |

## Why Niri is not the first command

The system does not need a compositor to update packages, verify its boot
chain, configure TRIM or zram, or establish a firewall. Those operations are
safer while the new installation still has few moving parts.

Niri and Kitty nevertheless appear early, immediately after the TTY baseline.
This avoids performing the entire workstation build in a bare console while
also ensuring that a graphical failure cannot block maintenance or recovery.

## General recommendations coverage

| ArchWiki area | Primary destination |
| --- | --- |
| System administration and package management | Chapters 01 and 02 |
| Booting and system maintenance | Chapters 01, 02, 03, and 12 |
| Graphical user interface | Chapters 05, 10, 11, and 13 |
| Power management and laptops | Chapter 06 |
| Multimedia | Chapter 07 |
| Networking and firewall | Chapter 04 |
| Input devices | Chapter 06 |
| Optimization and solid-state drives | Chapters 03 and 06 |
| System services | Chapters 02, 04, and 07 |
| Appearance | Chapters 08 and 10 |
| Console improvements | Chapters 02 and 08 |
| Applications | Chapter 09 |

Topics such as mail servers, public network shares, development toolchains,
virtualization, containers, and gaming remain optional extensions unless they
become part of the actual workstation requirements.

## Component selection rule

No desktop component is canonical merely because it appears in a draft. For
each role, the project compares maturity, maintenance activity, official-repo
availability, dependencies, native Wayland support, configuration format,
security implications, and recovery cost. The chosen components are then
documented as one coherent system instead of an unrelated package list.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [ArchWiki: Security](https://wiki.archlinux.org/title/Security)
- [ArchWiki: Laptop](https://wiki.archlinux.org/title/Laptop)
- [ArchWiki: Power management](https://wiki.archlinux.org/title/Power_management)
- [ArchWiki: Solid state drive](https://wiki.archlinux.org/title/Solid_state_drive)
- [ArchWiki: Niri](https://wiki.archlinux.org/title/Niri)
- [ArchWiki: List of applications](https://wiki.archlinux.org/title/List_of_applications)
