# 17 — Integrate Qt 6 appearance

## Goal

Extend the Midnight Circuit visual foundation to Qt 6 widget applications
without adding a KDE desktop stack or a second theme owner.

This chapter:

- installs `qt6ct` from the official Arch repository;
- deploys one reviewed Qt 6 configuration and color scheme from
  `niri-dotfiles`;
- makes Niri the sole owner of `QT_QPA_PLATFORMTHEME=qt6ct` for applications
  it launches;
- uses Qt's built-in Fusion widget style, Papirus Dark icons, and the existing
  Noto fonts;
- routes standard Qt dialogs through the XDG Desktop Portal provider;
- verifies native Wayland operation without forcing a global platform backend.

It does not configure Qt 5, install Kvantum or Plasma, set
`QT_STYLE_OVERRIDE`, force `QT_QPA_PLATFORM`, modify GTK, or add a systemd
service. Qt Quick applications and applications with their own themes may not
consume every QWidget palette choice; that is a toolkit boundary, not a reason
to stack another global override.

## Prerequisites

- Chapters 00 through 16 are complete.
- The chapter 15 visual foundation remains healthy.
- `niri-dotfiles` contains the reviewed chapter 17 commit and immutable tag
  `post-install-17-v1`.
- The dotfiles clone and current Stow deployment are clean.
- The Niri portal services passed their earlier verification.
- TTY3 remains available if the graphical session cannot be entered.

This procedure passed its first hardware validation on the canonical ThinkPad
on 2026-09-05.

Run the chapter as `neon` from Kitty. First select the exact dotfiles
checkpoint:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch --detach post-install-17-v1
git describe --tags --exact-match
git log -1 --oneline
niri validate --config niri/.config/niri/config.kdl
```

Stop on an unexpected local change, missing tag, or validation failure.
Detached HEAD is intentional while reproducing the chapter checkpoint.

`post-install-17-v1` is the tag **name** selected by `git switch`.
`Post-install chapter 17 Qt 6 appearance` is its annotated **message**, not a
second reference. Git reference names cannot contain spaces. Display both
fields when checking the checkpoint:

```bash
git tag --list 'post-install-17-v1'
git tag -n1 'post-install-17-v1'
```

## Fix the ownership boundary

The selected stack has one owner for each concern:

| Concern | Owner |
| --- | --- |
| Qt 6 widget style, palette, icons, fonts, and standard dialogs | qt6ct |
| Qt platform-theme selection inside this desktop | Niri `environment {}` |
| Wayland versus X11 platform plugin | Qt automatic selection |
| Portal implementation | Existing Niri/GNOME/GTK portal stack |
| GTK appearance | Chapter 15 GTK files and GSettings |
| Cursor | Niri plus the matching chapter 15 GSettings values |

Only `QT_QPA_PLATFORMTHEME=qt6ct` is added. Do not duplicate it in
`~/.bash_profile`, `/etc/environment`, an XDG autostart entry, or a systemd
user override. Niri passes its environment block to programs it starts,
including Fuzzel and the applications launched through Fuzzel.

`QT_QPA_PLATFORM` is deliberately absent. Current Qt can select its Wayland
platform plugin in the Niri session while retaining the X11/XWayland path for
an application that genuinely needs it. A one-application troubleshooting
override can be tested from a terminal later without changing the global
session.

## Audit existing Qt overrides

Before installing anything, inspect the current session and common persistent
locations:

```bash
printenv | grep -E '^QT_(QPA_PLATFORM|QPA_PLATFORMTHEME|STYLE_OVERRIDE)=' || true
grep -RnsE 'QT_(QPA_PLATFORM|QPA_PLATFORMTHEME|STYLE_OVERRIDE)' \
  ~/.bash_profile ~/.profile ~/.config/environment.d /etc/environment \
  2>/dev/null || true
pacman -Qq | grep -E '^(qt5ct|qt6ct|kvantum|kvantum-qt5)$' || true
```

An empty result is expected on the validated baseline. Stop and understand any
existing value before continuing. In particular, do not leave a previous
`QT_STYLE_OVERRIDE`, another `QT_QPA_PLATFORMTHEME`, or a forced platform in
place and then attempt to compensate inside qt6ct.

## Install the official package

Read current Arch news, then perform one complete transaction:

```bash
sudo pacman -Syu --needed qt6ct
sudo pacdiff --output
```

At the time this chapter was reviewed, Arch's `qt6ct` package depends on
`qt6-base` and `qt6-svg`. The current `qt6-base` package supplies both the
Wayland client platform plugin and the XDG Desktop Portal platform theme, so
the separate `qt6-wayland` package is not required by this client-only setup.
Always trust the current package metadata over this historical observation.

Verify the installed package and the exact plugin files:

```bash
pacman -Q qt6ct qt6-base qt6-svg
pacman -Qo /usr/bin/qt6ct
pacman -Qo /usr/lib/qt6/plugins/platformthemes/libqt6ct.so
pacman -Qo /usr/lib/qt6/plugins/platforms/libqwayland.so
pacman -Qo /usr/lib/qt6/plugins/platformthemes/libqxdgdesktopportal.so
```

All five commands must identify official packages. No service is enabled or
started by this installation.

If the transaction changed a boot component, verify the signed boot artifacts
before the next reboot:

```bash
sudo bootctl --esp-path=/boot list
sudo sbctl verify
```

## Review the dotfiles change

The chapter adds only these user-level responsibilities:

```text
niri/.config/niri/config.kdl
qt6ct/.config/qt6ct/qt6ct.conf
qt6ct/.config/qt6ct/colors/midnight-circuit.conf
```

Inspect them before deployment:

```bash
sed -n '1,80p' niri/.config/niri/config.kdl
sed -n '1,200p' qt6ct/.config/qt6ct/qt6ct.conf
sed -n '1,80p' qt6ct/.config/qt6ct/colors/midnight-circuit.conf
niri validate --config niri/.config/niri/config.kdl
```

The configuration uses:

| Setting | Selected value |
| --- | --- |
| Style | `Fusion`, supplied by Qt itself |
| Palette | Custom Midnight Circuit colors |
| Icon theme | `Papirus-Dark`, already installed in chapter 15 |
| General font | Noto Sans 10 |
| Fixed font | Noto Sans Mono 10 |
| Standard dialogs | `xdgdesktopportal` |

The qt6ct file format stores the color-scheme path as an absolute path. This
project's canonical account is `neon`, so the tracked value is
`/home/neon/.config/qt6ct/colors/midnight-circuit.conf` on both supported
ThinkPads. If the repository is reused under a different account, change and
review that one line before deployment; do not create a second hidden copy of
the palette.

Each of the three palette lines must contain 21 comma-separated Qt color roles.
Check that invariant without evaluating the file as shell code:

```bash
awk -F= '/^(active|disabled|inactive)_colors=/ {
  n=split($2, colors, /,[[:space:]]*/)
  printf "%s: %d colors\n", $1, n
  if (n != 21) bad=1
} END { exit bad }' qt6ct/.config/qt6ct/colors/midnight-circuit.conf
```

The result must report 21 colors for all three states.

## Preview and deploy with Stow

Check for pre-existing targets first:

```bash
test ! -e ~/.config/qt6ct/qt6ct.conf
test ! -e ~/.config/qt6ct/colors/midnight-circuit.conf
```

No output and exit status zero are expected on the validated baseline. If a
target already exists, inspect and back it up outside the active path; do not
use `stow --adopt`, because that can rewrite the repository with unreviewed
state.

Preview the new package and the Niri reconciliation:

```bash
stow --simulate --verbose --no-folding --target="$HOME" qt6ct
stow --restow --simulate --verbose --no-folding --target="$HOME" niri
```

When both previews are clean:

```bash
stow --verbose --no-folding --target="$HOME" qt6ct
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
readlink -f ~/.config/qt6ct/qt6ct.conf
readlink -f ~/.config/qt6ct/colors/midnight-circuit.conf
readlink -f ~/.config/niri/config.kdl
```

All three links must resolve inside
`~/Projects/CycloniteRDX/niri-dotfiles`.

## Start a fresh Niri session

The new environment variable affects newly spawned processes. Save work, exit
with `Super+Shift+E`, and log in again through tuigreet. A reboot is not
required.

In a new Kitty, verify the inherited session environment:

```bash
printf 'QT_QPA_PLATFORMTHEME=%s\n' "$QT_QPA_PLATFORMTHEME"
printenv QT_QPA_PLATFORM || true
printenv QT_STYLE_OVERRIDE || true
```

The first command must print `qt6ct`. The other two commands should print
nothing. A program launched by a systemd user unit would not automatically
inherit Niri's local environment block; no selected Qt desktop program in this
chapter uses that launch path.

Confirm that the portal broker remains available before asking Qt to use it:

```bash
systemctl --user is-active xdg-desktop-portal.service
systemctl --user --failed --no-pager
```

The broker must be active and no portal unit may be failed.

## Inspect the appearance without dirtying the repository

`qt6ct` is a configuration editor and can save window geometry or other UI
state when it closes. Because its live file is a Stow link into Git, use a
temporary copy for visual inspection:

```bash
mkdir -p ~/Documents/System-Records
qt_test_root=$(mktemp -d -p ~/Documents/System-Records qt6ct-test.XXXXXX)
mkdir "$qt_test_root/qt6ct"
cp -a ~/.config/qt6ct/. "$qt_test_root/qt6ct/"
XDG_CONFIG_HOME="$qt_test_root" QT_DEBUG_PLUGINS=1 \
  WAYLAND_DEBUG=client qt6ct 2>"$qt_test_root/qt.log"
```

Keep Kitty open while the qt6ct window is visible. Confirm:

1. no warning says that qt6ct is not selected as the platform theme;
2. the style is Fusion and the palette is Midnight Circuit;
3. normal, selected, disabled, tooltip, button, and input-field text remains
   readable;
4. Papirus Dark icons and both Noto font choices appear;
5. **Standard dialogs** is **XDG Desktop Portal**;
6. **Add** on the **Style Sheets** page opens a functioning portal file dialog
   that can be cancelled without changing the temporary configuration;
7. the window is sharp at the current scale and keyboard navigation works.

Close qt6ct, then prove that it used Wayland and that the real repository did
not receive its generated UI state:

```bash
grep -m1 'libqt6ct.so' "$qt_test_root/qt.log"
grep -m1 'libqwayland.so' "$qt_test_root/qt.log"
grep -m1 'libqxdgdesktopportal.so' "$qt_test_root/qt.log"
grep -m1 'wl_registry' "$qt_test_root/qt.log"
git status --short --branch
```

The trace must show the qt6ct platform theme, Wayland platform, XDG portal
theme, and a Wayland registry exchange. Git should still show a clean detached
checkpoint. Review the temporary evidence, then move the exact test directory
to Trash and clear the variable:

```bash
printf '%s\n' "$qt_test_root"
test -d "$qt_test_root"
gio trash "$qt_test_root"
unset qt_test_root
```

Do not launch the normal `qt6ct.desktop` entry merely to inspect values: that
edits the tracked live configuration. Use it only when intentionally changing
the reviewed theme and then inspect the Git diff before committing anything.

## Test real Qt 6 applications as they are added

The Qt6ct window proves the basic QWidget, Wayland, font, icon, and portal
path. For every future Qt 6 application, also test:

- text, menus, toolbars, disabled controls, selection, and focus contrast;
- open/save dialogs and portal parent-window behavior;
- internal and external displays, including fractional scale;
- native Wayland behavior and any application-specific XWayland fallback;
- whether the application is QWidget, Qt Quick, KDE Frameworks, sandboxed, or
  self-themed.

Do not install a second global theme engine because one application overrides
its own colors. Qt 5 does not read the Qt 6 configuration. Flatpak applications
may use their runtime's theme and portal integration rather than host plugin
files. Record those as application-specific boundaries.

## Rollback

Keep the current graphical session open while repairing a tracked file. To
remove the Qt 6 appearance links and restore the previous Niri environment:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --delete --verbose --target="$HOME" qt6ct
```

Then revert or repair the chapter 17 change in Git, validate, and reconcile
Niri:

```bash
niri validate --config niri/.config/niri/config.kdl
stow --restow --verbose --no-folding --target="$HOME" niri
```

Save work, exit Niri, and log in again so new processes no longer receive the
old platform-theme value. Removing the `qt6ct` package is optional after a
successful rollback; inspect the proposed transaction first:

```bash
sudo pacman -Rs qt6ct
```

Do not remove `qt6-base` or `qt6-svg` blindly: another installed application
may require them. If login becomes inaccessible, use TTY3 and the greetd
recovery path from chapter 11.

## Completion checklist

- [ ] `qt6ct`, `qt6-base`, and `qt6-svg` are official installed packages.
- [ ] The qt6ct, Wayland, and XDG portal plugin files have verified owners.
- [ ] No competing Qt theme or style override remains active.
- [ ] Niri validates and is the sole owner of `QT_QPA_PLATFORMTHEME=qt6ct`.
- [ ] `QT_QPA_PLATFORM` and `QT_STYLE_OVERRIDE` remain unset globally.
- [ ] Stow owns the qt6ct configuration and Midnight Circuit color scheme.
- [ ] All three palette states contain 21 roles.
- [ ] The temporary qt6ct test uses native Wayland and portal dialogs.
- [ ] Fusion controls, fonts, icons, selection, and disabled states are legible.
- [ ] GTK appearance, portals, locking, login, and other session components
      still work.
- [ ] Normal testing leaves the dotfiles repository clean.
- [ ] Qt 5, Kvantum, Plasma, a theme daemon, and a new service were not added.

## Sources

- [Arch package: qt6ct](https://archlinux.org/packages/extra/x86_64/qt6ct/)
- [Arch package files: qt6ct](https://archlinux.org/packages/extra/x86_64/qt6ct/files/)
- [Arch package files: qt6-base](https://archlinux.org/packages/extra/x86_64/qt6-base/files/)
- [qt6ct upstream](https://www.opencode.net/trialuser/qt6ct)
- [Qt Platform Abstraction](https://doc.qt.io/qt-6/qpa.html)
- [Qt `QGuiApplication` platform options](https://doc.qt.io/qt-6/qguiapplication.html#QGuiApplication)
- [Niri environment configuration](https://niri-wm.github.io/niri/Configuration:-Miscellaneous.html#environment)
- [XDG Desktop Portal design](https://flatpak.github.io/xdg-desktop-portal/docs/design-considerations.html)
