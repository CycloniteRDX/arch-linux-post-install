# 15 — Apply the visual foundation

## Goal

Give the modular Niri desktop one reproducible visual identity without turning
it into a monolithic shell. This chapter installs only official Arch packages,
deploys the reviewed `niri-dotfiles` theme and wallpaper files, and verifies the
result across GTK, Niri, Waybar, Fuzzel, Mako, swaylock, and Kitty.

The selected identity is **Midnight Circuit**: a dark navy and graphite base,
cyan as the primary accent, and fuchsia used sparingly. It keeps Noto Sans and
Noto Sans Mono and does not require a Nerd Font.

This chapter does not replace Mako with SwayNotificationCenter, add Eww, theme
Qt independently, enable a graphical greeter, or change lock and idle policy.
Those are separate, reversible decisions.

## Prerequisites

- Chapters 00 through 14 are complete.
- The machine passed the chapter 14 readiness gate.
- `niri-dotfiles` contains the reviewed chapter 15 commit.
- The repository and current Stow deployment are healthy.
- TTY3 remains available if a graphical configuration error occurs.

From Kitty:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch --detach post-install-15-v1
git describe --tags --exact-match
git log -1 --oneline
niri validate --config niri/.config/niri/config.kdl
```

Stop if Git reports an unexpected local change before switching or Niri rejects
the selected configuration. The checkpoint commands must identify
`post-install-15-v1` and commit `0e66f44`; detached HEAD is expected.

## Component policy

| Layer | Canonical choice | Reason |
| --- | --- | --- |
| GTK 3 | `adw-gtk3-dark` from `adw-gtk-theme` | Closely matches modern Adwaita/libadwaita applications without replacing their internals. |
| GTK 4/libadwaita | Standard `prefer-dark` color-scheme request | Respects application and libadwaita behavior instead of injecting unsupported CSS. |
| Icons | `Papirus-Dark` | Mature, broad coverage, and available in the official repository. |
| Cursor | `breeze_cursors`, 24 px | Complete, readable, and available in the official repository. |
| Fonts | Noto Sans 10 and Noto Sans Mono | Already installed and adequate without icon-font coupling. |
| Wallpaper | Project-owned `midnight-circuit.svg` | Reproducible, editable, attribution-free, and backed by a solid-color fallback. |

`adw-gtk-theme` is a GTK 3 theme. It does not rewrite GTK 4 applications or
libadwaita. The GTK 4 settings file therefore requests dark presentation,
icons, fonts, and cursor but deliberately omits `gtk-theme-name`.

Qt-specific theming remains deferred. Do not install `qt5ct`, `qt6ct`, Kvantum,
or an environment override merely to make one application resemble GTK.

## Install the official packages

Perform one complete transaction:

```bash
sudo pacman -Syu adw-gtk-theme papirus-icon-theme breeze-cursors
```

Verify the installed package ownership and exact theme directory names:

```bash
pacman -Q adw-gtk-theme papirus-icon-theme breeze-cursors
test -d /usr/share/themes/adw-gtk3-dark
test -f /usr/share/icons/Papirus-Dark/index.theme
test -f /usr/share/icons/breeze_cursors/index.theme
```

The names supplied to GTK, GSettings, and Niri are directory identifiers, not
display labels guessed from a settings application.

## Review the portable files

The dotfiles commit adds or updates only user-level appearance:

```text
.gitignore
theme/.config/gtk-3.0/settings.ini
theme/.config/gtk-4.0/settings.ini
wallpapers/.local/share/wallpapers/midnight-circuit.svg
wallpapers/.local/share/wallpapers/ATTRIBUTION.md
niri/.config/niri/config.kdl
waybar/.config/waybar/style.css
fuzzel/.config/fuzzel/fuzzel.ini
mako/.config/mako/config
swaylock/.config/swaylock/config
kitty/.config/kitty/kitty.conf
```

The `.gitignore` change keeps ignoring personal files whose names end in
`.local`, but explicitly allows the XDG directory used by Stow packages such as
`wallpapers/.local/`. Without that distinction, Git would silently omit the
wallpaper package even though the files existed in the working tree.

Inspect the files before deployment:

```bash
sed -n '1,160p' theme/.config/gtk-3.0/settings.ini
sed -n '1,160p' theme/.config/gtk-4.0/settings.ini
sed -n '1,220p' wallpapers/.local/share/wallpapers/ATTRIBUTION.md
niri validate --config niri/.config/niri/config.kdl
```

The wallpaper is authored for this project and dedicated under CC0 1.0. It is
an SVG rather than a downloaded binary whose provenance could later be lost.
Niri starts swaybg with the image when readable and falls back to `#0b0f17`
when it is missing or inaccessible.

## Record the current GSettings values

On Wayland, GTK can read keys present in `org.gnome.desktop.interface` from
GSettings instead of `settings.ini`. Keep the tracked files for reconstruction,
but also set the session-facing database explicitly.

First confirm the schema exists and record the current values in the private
system record created in chapter 14:

```bash
gsettings list-schemas | grep -qx 'org.gnome.desktop.interface'
gsettings get org.gnome.desktop.interface color-scheme
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface icon-theme
gsettings get org.gnome.desktop.interface cursor-theme
gsettings get org.gnome.desktop.interface cursor-size
gsettings get org.gnome.desktop.interface font-name
```

These values make rollback exact. Do not commit the private system record.

## Preview and deploy with Stow

The `theme` package is new. The existing visual packages are reconciled in one
reviewed operation:

```bash
stow --simulate --verbose --no-folding --target="$HOME" theme
stow --restow --simulate --verbose --no-folding --target="$HOME" niri waybar fuzzel mako wallpapers swaylock kitty
```

Stop on any conflict. In particular, do not use `--adopt` on existing GTK
settings because that could rewrite the repository with unreviewed local state.

When the preview is clean:

```bash
stow --verbose --no-folding --target="$HOME" theme
stow --restow --verbose --no-folding --target="$HOME" niri waybar fuzzel mako wallpapers swaylock kitty
niri validate
```

Confirm the important targets resolve inside the clone:

```bash
readlink -f ~/.config/gtk-3.0/settings.ini
readlink -f ~/.config/gtk-4.0/settings.ini
readlink -f ~/.config/niri/config.kdl
readlink -f ~/.local/share/wallpapers/midnight-circuit.svg
```

## Apply the Wayland-facing preferences

Set the same reviewed values in GSettings:

```bash
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
gsettings set org.gnome.desktop.interface gtk-theme 'adw-gtk3-dark'
gsettings set org.gnome.desktop.interface icon-theme 'Papirus-Dark'
gsettings set org.gnome.desktop.interface cursor-theme 'breeze_cursors'
gsettings set org.gnome.desktop.interface cursor-size 24
gsettings set org.gnome.desktop.interface font-name 'Noto Sans 10'
```

Check the effective values:

```bash
gsettings get org.gnome.desktop.interface color-scheme
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface icon-theme
gsettings get org.gnome.desktop.interface cursor-theme
gsettings get org.gnome.desktop.interface cursor-size
gsettings get org.gnome.desktop.interface font-name
```

Niri's `cursor` block also sets `XCURSOR_THEME` and `XCURSOR_SIZE` for the
session it launches. Keeping the compositor and GSettings values identical
prevents different cursors over native Wayland surfaces and GTK windows.

## Start one fresh session

Niri can reload much of its configuration live, but cursor environment and
every newly launched application are best tested from a fresh session. Save
work, exit Niri through `Super+Shift+E`, and log in again through tuigreet. A
reboot is not required.

After login:

```bash
printf 'XCURSOR_THEME=%s\nXCURSOR_SIZE=%s\n' "$XCURSOR_THEME" "$XCURSOR_SIZE"
pgrep -xc niri
pgrep -xc waybar
pgrep -xc mako
pgrep -xc swaybg
pgrep -xc swayidle
git -C ~/Projects/CycloniteRDX/niri-dotfiles status --short --branch
```

The cursor values must be `breeze_cursors` and `24`. Each session component
must have exactly one process, and normal startup must not modify tracked files.

## Visual and functional test

Check behavior, not only screenshots:

1. Confirm the SVG fills the internal display without distortion or a visible
   solid-color flash that persists.
2. Open Nautilus, Loupe, Papers, GNOME Calendar, GNOME Text Editor, and Firefox.
   GTK 3 and GTK 4/libadwaita applications must remain readable in dark mode.
3. Open Fuzzel and confirm Papirus icons appear for the installed applications.
4. Check focused, inactive, urgent, muted, warning, and session states in
   Waybar where those states are available.
5. Send low, normal, and critical test notifications:

   ```bash
   notify-send --urgency=low 'Midnight Circuit' 'Low urgency test'
   notify-send --urgency=normal 'Midnight Circuit' 'Normal urgency test'
   notify-send --urgency=critical 'Midnight Circuit' 'Critical urgency test'
   ```

6. Lock with `Super+Alt+L`; verify prompt, verification, error, and unlock
   states remain legible.
7. Open Kitty and verify ANSI colors, selection, URLs, tabs, and ordinary text.
8. Move the cursor across the wallpaper, GTK applications, Firefox, Kitty,
   Waybar, Fuzzel, and the lock screen. Its theme and size must not change.
9. If an external display is available, connect it once and check wallpaper
   fill and cursor readability on both outputs. Do not add a host-specific mode
   to solve an appearance issue.

A visual mismatch in one toolkit is diagnosed at that toolkit boundary. Do not
paper over it with global `GTK_THEME`, untracked CSS, or a second theme daemon.

## Rollback

Keep the current graphical session open while repairing a bad tracked file. Use
Git to inspect or revert the chapter 15 commit, then reconcile only the affected
Stow packages:

```bash
niri validate --config niri/.config/niri/config.kdl
stow --restow --verbose --no-folding --target="$HOME" niri waybar fuzzel mako wallpapers swaylock kitty
```

Remove the GTK preference links without deleting repository files with:

```bash
stow --delete --verbose --target="$HOME" theme
```

Restore each GSettings key to the value recorded before this chapter. Do not
blindly run `gsettings reset` if the account already had a deliberate value.
The solid wallpaper fallback means deleting or withholding the SVG does not
prevent Niri from starting.

If the graphical session cannot be entered, use TTY3, validate the tracked Niri
file, and disable greetd temporarily as documented in chapter 11.

## Completion checklist

- [ ] All three appearance packages came from official Arch repositories.
- [ ] Niri validates before and after deployment.
- [ ] Stow owns the two GTK settings files and the project wallpaper.
- [ ] GSettings and Niri agree on the cursor name and size.
- [ ] GTK 3, GTK 4/libadwaita, Firefox, and Kitty remain readable.
- [ ] Papirus icons appear in Fuzzel and GTK applications.
- [ ] Waybar, Mako, swaylock, Fuzzel, Kitty, and the wallpaper share the reviewed palette.
- [ ] Every session component starts exactly once after a fresh login.
- [ ] The dotfiles repository remains clean after normal use.
- [ ] No AUR theme, Nerd Font, Qt override, new service, or new desktop shell was introduced.

## Sources

- [Arch package: adw-gtk-theme](https://archlinux.org/packages/extra/any/adw-gtk-theme/)
- [Arch package: papirus-icon-theme](https://archlinux.org/packages/extra/any/papirus-icon-theme/)
- [Arch package: breeze-cursors](https://archlinux.org/packages/extra/x86_64/breeze-cursors/)
- [ArchWiki: GTK](https://wiki.archlinux.org/title/GTK)
- [ArchWiki: Icons](https://wiki.archlinux.org/title/Icons)
- [ArchWiki: Cursor themes](https://wiki.archlinux.org/title/Cursor_themes)
- [Niri cursor configuration](https://niri-wm.github.io/niri/Configuration:-Miscellaneous.html#cursor)
