# 00 — Post-installation profile

## Goal

Define the exact system that this repository will build from the first working
TTY. Later chapters use these choices as fixed assumptions and introduce
alternatives only when a component has not yet been selected.

This chapter does not modify the computer.

## Supported target

| Item | Canonical value |
| --- | --- |
| Computer | Lenovo ThinkPad T14 Gen 1 AMD |
| Processors | AMD Ryzen 5 PRO 4650U or Ryzen 7 PRO 4750U |
| Graphics | Integrated AMD Radeon graphics using the `amdgpu` kernel driver |
| Memory | 16 GiB or more |
| Internal drive | One approximately 512 GB NVMe SSD |
| Installed system | Current Arch Linux from `arch-linux-runbook` |
| Regular user | `neon` with `wheel` membership and working `sudo` |
| Primary use | Personal laptop and daily-driver workstation |

The procedure may be adapted to another machine, but firmware, graphics,
power, input, storage, and Secure Boot assumptions must be reviewed first.

## Required starting state

The post-install procedure begins only after the runbook completion checks have
passed:

- UEFI starts signed systemd-boot through `Linux Boot Manager`.
- Secure Boot is enabled in User Mode with owner and Microsoft certificates.
- Normal and fallback UKIs are present, signed, and bootable.
- `/dev/nvme0n1p2` is a LUKS2 container opened as `cryptlvm`.
- LVM volume group `vg0` provides `root`, `home`, and `swap`.
- Root and home use ext4.
- The EFI System Partition is mounted at `/boot` with
  `fmask=0177,dmask=0077`.
- The encrypted 16 GiB disk swap is active.
- Hibernation and `resume=` are not configured.
- NetworkManager and systemd-timesyncd are enabled.
- `neon` can log in at a TTY and run password-protected `sudo` commands.
- No graphical environment is required to repair the system.

Chapter 01 will audit this state again rather than assuming that every item is
still true.

## Operator and privilege model

- Log in as `neon`; do not use a persistent root login for ordinary work.
- Run unprivileged inspection commands without `sudo`.
- Use `sudo` only for the command that requires administrative access.
- Read package transactions and service changes before confirming them.
- Do not paste passwords, tokens, recovery keys, or Wi-Fi secrets into command
  arguments or repository files.
- Do not enable a service merely because its package was installed.

Installing the OpenSSH package provides both client and server programs, but
does not by itself expose an SSH service. `sshd.service` remains disabled unless
remote login becomes an explicit, separately secured requirement.

## Package-management policy

- Use `pacman` and official Arch repositories for the canonical system.
- Perform complete upgrades with `pacman -Syu`; partial upgrades are unsupported.
- Check Arch news for manual interventions before routine upgrades.
- Keep the mirror policy simple and retain a known-working mirror list.
- Use package cache retention deliberately rather than deleting every old
  package immediately.
- Install `git` early so the documentation and dotfiles repositories can be
  obtained from the TTY system.
- Install `base-devel` only when building packages becomes necessary.
- Treat the AUR as untrusted user-contributed build instructions: inspect the
  PKGBUILD and related files before building.
- Do not bootstrap software with an unreviewed `curl ... | sh` command.
- An AUR helper is optional and is not a prerequisite for the workstation.
  When chapter 16 is deliberately applied, Paru is the selected helper after
  one complete manual review and build cycle.

Detailed pacman, Git, PKGBUILD, and AUR usage belongs in the handbook.

## Storage and memory policy

- Keep ext4 for both `/` and `/home`; this repository does not migrate to
  Btrfs or introduce snapshot-based rollback.
- Keep the intentional unpartitioned SSD tail and the 256 MiB of free space in
  `vg0` unchanged.
- Do not add continuous `discard` to the ext4 mount records.
- Enable discard propagation through the LUKS mapping so periodic TRIM can
  reach the NVMe device.
- Enable `fstrim.timer` and verify the complete block-device path.
- Add zram with a higher swap priority than the disk-backed LV.
- Retain the encrypted disk swap as a lower-priority fallback.
- Do not configure hibernation or a resume device.
- Introduce memory-tuning values only when their purpose and measured behavior
  are documented.

The accepted LUKS discard policy reveals which encrypted regions are currently
unused. It does not reveal the contents of allocated blocks. This trade-off is
accepted for SSD maintenance on the canonical personal-laptop threat model.

## Security and network policy

- NetworkManager remains the only network-management service.
- systemd-timesyncd remains the initial time-synchronization service.
- Firewalld is the canonical firewall because its zones integrate cleanly with
  NetworkManager on a laptop that moves between trusted and untrusted networks.
- Incoming connections are denied unless a later, documented service requires
  an explicit exception.
- Listening sockets and enabled services are audited before and after adding
  workstation components.
- Remote SSH login, network shares, discovery services, and remote desktop are
  disabled unless intentionally configured.
- DNS continues to use the active network's configuration initially. Encrypted
  DNS and local resolver choices require a separate design decision.
- Secure Boot keys remain only on the encrypted root filesystem and in a
  protected offline backup.
- LUKS and Secure Boot recovery material is never stored in the ESP or in Git.

This is a practical workstation baseline, not an attempt to maximize every
hardening control regardless of usability.

## Laptop, power, and firmware policy

- Use `fwupd` for supported firmware updates and verify that the machine is
  recognized before applying an update.
- Preserve normal suspend and test both manual suspend and lid-close behavior.
- Keep hibernation disabled.
- Use only one high-level power-policy manager.
- Do not enable TLP and power-profiles-daemon together.
- Select the canonical power manager only after checking driver support and
  behavior on both Ryzen generations.
- Prefer kernel and firmware defaults before adding fan-control or aggressive
  power-tuning daemons.
- Treat battery thresholds, TrackPoint behavior, function keys, external
  displays, and docking as ThinkPad-specific integration tests.

## Graphical foundation

| Role | Canonical direction |
| --- | --- |
| Compositor | Niri from the official Arch repository |
| Terminal | Kitty |
| Graphics | `amdgpu`, Mesa, and Vulkan support appropriate to the AMD iGPU |
| X11 compatibility | `xwayland-satellite` |
| Desktop portals | Niri-compatible GNOME and GTK portal backends |
| Audio session | PipeWire with WirePlumber |
| Privilege authorization | Polkit plus one graphical authentication agent |
| Login start | Manual `niri-session` first; greetd only after validation |
| Desktop shell | Individually selected components; no complete-shell dependency |
| Qt 6 appearance | qt6ct with Fusion, Midnight Circuit, Papirus Dark, Noto fonts, and portal dialogs |

Niri and Kitty are installed after the initial TTY maintenance, storage, and
security baseline. Their system packages belong here; the actual Niri and Kitty
configuration belongs in `niri-dotfiles`.

Chapter 05 coordinates a minimal bootstrap from that repository containing
only the bindings needed to open Kitty and exit Niri. It does not duplicate the
configuration in this repository. The complete, themed configuration is
deployed during the later dotfiles handoff.

The first manual session must prove that Kitty opens, input works, the correct
display mode is active, applications can exit, and the user can return to TTY.
No greeter is enabled until that path is reliable.

## Early-boot presentation policy

The runbook and the frozen `v1.0.0` baseline remain deliberately textual so a
new installation exposes its real LUKS, initramfs, and service boundaries.
Chapter 19 may add Plymouth only after both signed UKIs and the recovery path
are proven.

After that extension:

- the normal UKI includes Plymouth and embeds `quiet splash`;
- the fallback UKI omits Plymouth and both presentation parameters;
- both retain the same LUKS UUID, discard, zswap, root, and read-write policy;
- systemd-boot keeps its three-second menu and disabled editor;
- `sd-encrypt`, not Plymouth, remains responsible for LUKS unlocking;
- the official `bgrt` theme is the first baseline;
- TPM2 unlock and custom boot artwork remain later, separate changes.

Plymouth does not install user configuration and creates no new dotfiles
checkpoint.

## Desktop-component policy

The final workstation will provide the functions expected from a complete
desktop environment, but each role is selected independently:

- greeter and session launch;
- application launcher;
- status bar or shell UI;
- notifications;
- lock screen and idle handling;
- logout, reboot, and shutdown interface;
- wallpaper management;
- screenshots, screen recording, and clipboard history;
- file management and removable-media handling;
- secrets and authentication prompts;
- GTK and Qt dark themes;
- audio, brightness, network, Bluetooth, and power controls.

Component selection must consider maturity, maintenance, official-repository
availability, native Wayland behavior, dependency size, configuration quality,
security implications, and ease of recovery. A visually attractive result is a
goal, but it does not override reliability on a daily driver.

Qt 6 appearance is owned by qt6ct after chapter 17. Niri exports only
`QT_QPA_PLATFORMTHEME=qt6ct`; Qt chooses its platform backend automatically.
Qt 5, Kvantum, Plasma, `QT_STYLE_OVERRIDE`, and a global forced
`QT_QPA_PLATFORM` are not part of the profile.

After chapter 18, the existing swayidle process remains the only session-idle
coordinator. It locks at five minutes, powers monitors off at ten, and calls a
fail-closed helper at 30 minutes. That helper requests suspend only when
UPower reports battery operation and tells systemd to honor active inhibitors.
It does not create automatic suspend on AC or after the Niri session ends.

## Applications policy

The completed system will include, at minimum:

- a web browser;
- Kitty and console utilities;
- a graphical file manager;
- PDF and image viewers;
- a text editor and a later Vim learning environment;
- archive and compression tools;
- audio and video playback;
- screenshot and screen-recording tools;
- password or secrets integration;
- Git and development utilities needed by the user's projects.

Exact applications are selected in their own chapter instead of installing a
large undifferentiated package list.

## User directories, configuration, and secrets

- Create standard XDG user directories explicitly.
- Keep the system language in English while allowing either US or Spanish
  physical-keyboard layouts per machine.
- Use a consistent dark appearance across GTK and Qt applications where
  practical.
- Store reviewed user configuration in `niri-dotfiles`.
- Use GNU Stow or an equally transparent symlink deployment only after the
  repository layout is reviewed.
- Never commit SSH private keys, sbctl keys, LUKS material, browser profiles,
  password databases, tokens, Wi-Fi connections, machine IDs, logs, or caches.

## Recovery and backup policy

Before the workstation is considered complete:

- Preserve the LUKS passphrase independently of the laptop.
- Create and protect a LUKS header backup.
- Create an encrypted offline backup of the sbctl keys.
- Retain a verified Arch installation medium and written chroot recovery steps.
- Define a user-data backup destination and test restoring at least one file.
- Keep the TTY and manual Niri launch paths available after enabling greetd.
- Record enough installed-package and enabled-service state to reconstruct the
  machine without copying runtime-generated files.

Snapshots, sync, and backups solve different problems; none is treated as a
substitute for the others.

## Completion criteria

The post-install project is complete only when:

- package upgrades and required maintenance can be performed safely;
- TRIM reaches the NVMe device and zram precedes disk swap;
- the firewall is active and exposed services are understood;
- firmware, suspend, lid behavior, battery policy, and input devices work;
- manual and greetd-started Niri sessions both work without losing TTY access;
- audio, Bluetooth, removable media, portals, clipboard, screenshots, lock,
  idle, and logout paths are verified;
- the required daily applications are installed and usable;
- user configuration can be redeployed from `niri-dotfiles` without secrets;
- backups and recovery steps have been tested;
- no unexplained system or user units are failed.

## Out of scope

The canonical path does not configure hibernation, TPM-based automatic LUKS
unlocking, public servers, remote SSH access, network shares, virtualization,
containers, gaming, or a complete development toolchain. These can be added as
optional extensions when a real requirement exists.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [ArchWiki: Security](https://wiki.archlinux.org/title/Security)
- [ArchWiki: Laptop](https://wiki.archlinux.org/title/Laptop)
- [ArchWiki: Power management](https://wiki.archlinux.org/title/Power_management)
- [ArchWiki: Solid state drive](https://wiki.archlinux.org/title/Solid_state_drive)
- [ArchWiki: Niri](https://wiki.archlinux.org/title/Niri)
- [ArchWiki: List of applications](https://wiki.archlinux.org/title/List_of_applications)

## Next step

Continue with chapter 01 to audit the handoff from `arch-linux-runbook` before
installing or changing any post-install component.
