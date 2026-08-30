# 10 — Integrate desktop components

## Goal

Turn the verified Niri bootstrap into a usable but still recoverable desktop by
adding one reviewed component for each visible desktop role:

- Waybar for status and workspace information;
- Fuzzel as the application launcher;
- Mako as the notification daemon;
- swaybg for the background;
- Niri's built-in screenshot interface;
- reproducible configuration from `niri-dotfiles`.

This chapter deliberately does **not** add a greeter, automatic graphical
login, screen locker, idle daemon, suspend automation, shutdown menu, theme
framework, or desktop shell. Login, lock, idle and suspend form one lifecycle
and remain in chapter 11.

## Prerequisites

- Chapters 01 through 09 are complete.
- `niri-session -l` starts successfully from TTY.
- Kitty opens with `Super+Enter`.
- PipeWire, WirePlumber, NetworkManager and the notification-dependent calendar
  backend work as documented earlier.
- The local `niri-dotfiles` clone is clean and contains the chapter 10 files.

Run package and inspection commands in Kitty inside Niri unless a step says to
return to TTY.

## Component boundary

| Role | Component | Boundary |
| --- | --- | --- |
| Compositor | Niri | Windows, workspaces, key bindings and screenshots. |
| Status UI | Waybar | Read-only status plus narrowly scoped click/scroll actions. |
| Launcher | Fuzzel | Launch installed desktop applications; no shell or file indexing. |
| Notifications | Mako | Implements the notification service and renders transient notifications. |
| Background | swaybg | Draws one image or solid colour; it is not a theme manager. |
| Session exit | Niri confirmation dialog | Ends only the graphical session; no direct power-off or reboot button. |

These programs are complementary. Mako is not a portal, Fuzzel is not a shell,
and Waybar does not own networking, audio, power management or Bluetooth. It
shows state exposed by the corresponding system or user services.

## Audit the current session

Confirm that no alternative components are already active:

```bash
pgrep -a waybar
pgrep -a mako
pgrep -a swaybg
pgrep -a dunst
pgrep -a swaync
```

No output is expected from the clean bootstrap. If an alternative bar,
notification daemon or wallpaper process is present, stop and identify where
it starts. Do not run two implementations of the same session role.

Inspect the current Stow packages and targets:

```bash
cd ~/Documents/Repositories/niri-dotfiles
git status --short --branch
find ~/.config -maxdepth 2 -type f \( -path '*/waybar/*' -o -path '*/fuzzel/*' -o -path '*/mako/*' \) -print
```

Adjust only the repository path if the clone is stored elsewhere. Existing
untracked configuration must be reviewed and backed up before Stow is used.

## Install the selected components

Read current Arch News, then perform one complete transaction:

```bash
sudo pacman -Syu waybar fuzzel mako swaybg libnotify
```

`libnotify` supplies `notify-send` for verification. Niri already provides its
interactive, output and window screenshot actions, so this chapter does not
install a second screenshot frontend. `wl-clipboard`, `brightnessctl`,
`lm_sensors`, PipeWire and WirePlumber were installed in earlier chapters.

Verify the commands:

```bash
command -v waybar fuzzel mako swaybg notify-send wpctl brightnessctl
pacman -Q waybar fuzzel mako swaybg libnotify
```

## Validate before deployment

From the `niri-dotfiles` root, inspect the exact files:

```bash
git status --short --branch
sed -n '1,240p' niri/.config/niri/config.kdl
sed -n '1,260p' waybar/.config/waybar/config.jsonc
sed -n '1,220p' waybar/.config/waybar/style.css
sed -n '1,160p' fuzzel/.config/fuzzel/fuzzel.ini
sed -n '1,160p' mako/.config/mako/config
```

Validate Niri while the current known-good session is still running:

```bash
niri validate --config niri/.config/niri/config.kdl
```

An exit status of zero is required. Waybar's JSONC is parsed when Waybar starts;
the controlled manual launch below is its functional validation.

## Deploy with GNU Stow

Preview every new package first:

```bash
stow --simulate --verbose --no-folding --target="$HOME" waybar fuzzel mako wallpapers
```

Stop on any conflict. Do not use `--adopt`, delete an existing file blindly, or
turn a real user configuration into an unreviewed repository change.

Deploy the new packages and reconcile the existing Niri package:

```bash
stow --verbose --no-folding --target="$HOME" waybar fuzzel mako wallpapers
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
```

The initial background is intentionally the solid colour `#18181c`. A personal
wallpaper can be added later by replacing the `swaybg` spawn arguments with a
reviewed path under `~/.local/share/wallpapers/`.

## Test components without restarting Niri

Launch each component manually from Kitty:

```bash
waybar
```

Confirm that the bar appears and that the terminal remains occupied by the
process. Inspect any warnings, press `Ctrl+C`, then test:

```bash
fuzzel
mako
```

Fuzzel should open and can be dismissed with `Escape`. Mako remains in the
foreground; from a second Kitty window run:

```bash
notify-send "Mako test" "Notifications are working"
```

The notification must appear once. Stop the foreground Mako process with
`Ctrl+C`. Test the background similarly:

```bash
swaybg -c '#18181c'
```

After the colour appears, stop it with `Ctrl+C`.

## Restart the graphical session once

Save work, press `Super+Shift+E`, accept Niri's confirmation dialog, and return
to TTY. Start the reviewed session again:

```bash
niri-session -l
```

Waybar, Mako and swaybg must now start exactly once:

```bash
pgrep -a waybar
pgrep -a mako
pgrep -a swaybg
```

Each command should show one process. If a component is duplicated, find the
second autostart source instead of hiding the symptom with `pkill`.

## Functional verification

Test these interactions:

1. `Super+D` opens Fuzzel and launches Firefox.
2. `notify-send "Desktop test" "Mako is active"` displays one notification.
3. Create a short local GNOME Calendar event and verify its reminder reaches
   Mako.
4. `Print` opens Niri's interactive screenshot UI.
5. `Ctrl+Print` captures the focused output; `Alt+Print` captures the focused
   window.
6. Confirm new screenshots appear under the configured XDG Pictures directory
   and can also be pasted from the clipboard.
7. Audio, microphone and brightness keys update the corresponding devices.
8. Waybar accurately reports network, battery, clock, CPU, RAM and temperature.
9. Clicking `Exit` opens Niri's confirmation dialog; cancel it during testing.

The bar deliberately uses ordinary text labels rather than Nerd Font glyphs.
This avoids undocumented font dependencies. Styling and icon experiments can
be added later without changing the component architecture.

## Recovery

If the graphical session becomes unusable, switch to another TTY with
`Ctrl+Alt+F3`, log in, and validate:

```bash
niri validate
journalctl --user -b --no-pager | tail -n 200
```

Disable only the newly added links from the dotfiles repository:

```bash
stow --delete --verbose --target="$HOME" waybar fuzzel mako wallpapers
```

To restore the previous Niri bootstrap, use Git to inspect and deliberately
revert the chapter 10 change in the repository, then restow `niri`. Do not
delete the repository or overwrite `~/.config/niri/config.kdl` with an
untracked copy.

## Completion checklist

- [ ] The five packages are installed from official repositories.
- [ ] `niri validate` succeeds before and after deployment.
- [ ] Waybar, Mako and swaybg each run exactly once after a fresh session start.
- [ ] Fuzzel launches installed applications.
- [ ] Calendar and manual notifications appear through Mako.
- [ ] Niri's three screenshot flows work without an additional frontend.
- [ ] Audio, microphone and brightness controls work.
- [ ] No greeter, locker, idle daemon or suspend automation was introduced.
- [ ] TTY recovery remains available.

## Sources

- [Niri key bindings and screenshot actions](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Key-Bindings)
- [Waybar Niri workspaces module](https://man.archlinux.org/man/waybar-niri-workspaces.5.en)
- [Waybar manual](https://man.archlinux.org/man/waybar.5.en)
- [Arch package search](https://archlinux.org/packages/)
