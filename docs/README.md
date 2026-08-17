# Post-install roadmap

The sequence is organized around usable checkpoints. Each stage should leave
the machine in a known-good state before the next one begins.

| Stage | Outcome |
| --- | --- |
| 00. Handoff audit | Confirm the runbook result, package state, boot path, network, time, and recovery access. |
| 01. Maintenance baseline | Establish full-upgrade habits, mirror policy, cache policy, logs, and routine maintenance. |
| 02. Storage and memory | Configure periodic TRIM, document encrypted discard, enable zram, retain low-priority disk swap, and verify behavior. |
| 03. Security baseline | Install and configure the chosen firewall, review exposed services, apply practical hardening, and document recovery. |
| 04. ThinkPad integration | Configure firmware updates, battery and power policy, thermals, hardware monitoring, suspend, and laptop keys. |
| 05. Core workstation services | Configure graphics, audio, Bluetooth, printing, removable media, polkit, portals, and secret storage. |
| 06. User environment | Create XDG user directories and install fonts, terminal tools, shell utilities, and accessibility essentials. |
| 07. Daily applications | Install and verify the browser, terminal, file manager, PDF reader, archive tools, media applications, and development tools. |
| 08. Niri foundation | Install Niri and the Wayland/session dependencies required for a reliable first graphical login. |
| 09. Desktop components | Select and integrate greetd, launcher, status UI, notifications, lock, idle, screenshots, wallpaper, themes, and logout. |
| 10. Dotfiles handoff | Deploy reviewed configuration from `niri-dotfiles` without copying secrets or machine state. |
| 11. Daily-driver verification | Test startup, shutdown, suspend, networking, media, external displays, portals, updates, recovery, and backups. |

## Component selection rule

No desktop component is canonical merely because it appears in a draft. For
each role, the project will compare maturity, maintenance activity,
dependencies, Wayland integration, configuration format, and recovery cost.
The chosen stack will then be documented as one coherent system.
