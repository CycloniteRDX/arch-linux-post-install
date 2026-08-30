# 13 — Complete the dotfiles handoff

## Goal

Replace the deliberately tiny Niri bootstrap with the reviewed portable
daily-driver configuration and prove that a clean clone of `niri-dotfiles` can
reconstruct the entire user-level desktop without copying generated state or
secrets.

This chapter adds the remaining portable Niri behavior and a Kitty package. It
does not hard-code an output, refresh rate, scale, wallpaper image, hostname,
username path, credential, browser profile, or application cache.

## Prerequisites

- Chapters 01 through 12 are complete.
- The recovery bundle and first Restic restore test succeeded.
- greetd can be disabled from TTY3 if the graphical session fails.
- The `niri-dotfiles` clone is clean before the chapter 13 files are copied.
- Every existing Stow package from chapters 05 through 11 is deployed.

Run these checks from Kitty:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
stow --simulate --verbose --no-folding --target="$HOME" niri autostart mimeapps waybar fuzzel mako wallpapers swaylock
niri validate
```

The Git status must be clean before applying the new commit. Stow may report
that existing links are already owned; it must not report a real conflict.

## Portable policy

The final shared configuration fixes only behavior that belongs on both
ThinkPads:

| Area | Portable decision |
| --- | --- |
| Keyboard | US layout, 300 ms repeat delay, 25 repeats/s |
| Touchpad | Tap-to-click and disable-while-typing enabled |
| TrackPoint | Niri/libinput defaults; no guessed acceleration curve |
| Focus | Explicit keyboard focus; no focus-follows-mouse |
| Workspaces | Dynamic Niri workspaces, navigated vertically |
| Columns | Presets at one third, one half, and two thirds |
| Windows | Vertical stacking inside columns plus floating/tabbed toggles |
| Appearance | 12 px gaps, dark background, cyan-to-fuchsia focus ring |
| Terminal | Kitty with Noto Sans Mono and matching opaque palette |
| Output | Preferred automatic mode and scale until measured per host |

Natural scrolling is not enabled silently. TrackPoint acceleration and scroll
button behavior are also left at libinput defaults until tested physically.

## Why outputs remain automatic

The first T14 has reported `eDP-1` modes near
`1920x1080@60.049` and `1920x1080@48.040`, and scale 1.5 has previously been a
useful experiment. Those values are evidence for that panel, not a portable
contract for both laptops.

Niri requires an exact mode string, including its three decimal refresh digits.
A shared active output block can therefore reject a replacement panel or the
second ThinkPad's slightly different timing. The portable config omits
`output {}` so Niri selects the preferred mode. Chapter 14 will record each
machine's actual output report before host overrides are designed.

## Review the complete Niri configuration

From the dotfiles root:

```bash
sed -n '1,320p' niri/.config/niri/config.kdl
niri validate --config niri/.config/niri/config.kdl
```

Validation must exit successfully. The `binds {}` section is intentionally
complete because Niri does not merge omitted default bindings into a user
configuration.

The shortcut families are:

| Purpose | Keys |
| --- | --- |
| Help | `Super+Shift+/` |
| Terminal / launcher / overview | `Super+Enter`, `Super+D`, `Super+O` |
| Focus | `Super` + arrows or `H/J/K/L` |
| Move | `Super+Ctrl` + arrows or `H/J/K/L` |
| Workspaces | `Super+PageUp/PageDown` |
| Send column to workspace | `Super+Ctrl+PageUp/PageDown` |
| Reorder workspace | `Super+Shift+PageUp/PageDown` |
| Width / height presets | `Super+R`, `Super+Shift+R` |
| Maximize / fullscreen / center | `Super+F`, `Super+Shift+F`, `Super+C` |
| Floating / tabbed | `Super+V`, `Super+Shift+V`, `Super+W` |
| Close | `Super+Q` |
| Lock / exit | `Super+Alt+L`, `Super+Shift+E` |

The mouse wheel with `Super` switches vertical workspaces. Adding Shift moves
focus horizontally between columns. Native three-finger Niri gestures remain
available because the configuration does not replace them.

## Review Kitty

```bash
sed -n '1,220p' kitty/.config/kitty/kitty.conf
kitty --config kitty/.config/kitty/kitty.conf --debug-config
```

The baseline deliberately uses Noto Sans Mono rather than a Nerd Font. It keeps
the background opaque, disables the audible bell, retains 10,000 scrollback
lines, and defines explicit clipboard/window/tab shortcuts. Shell aliases,
prompt frameworks, SSH keys, history, and per-project environment variables do
not belong in Kitty's portable package.

## Preview the handoff

The Niri target is already a Stow link, while Kitty is new. Preview both:

```bash
stow --simulate --verbose --no-folding --target="$HOME" kitty
stow --restow --simulate --verbose --no-folding --target="$HOME" niri
```

If `~/.config/kitty/kitty.conf` already exists as a real file, stop and inspect
it. Back it up outside the active path; do not use `--adopt`, because that would
rewrite the repository with unreviewed local content.

## Deploy and validate

Deploy Kitty and reconcile the Niri link:

```bash
stow --verbose --no-folding --target="$HOME" kitty
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
readlink -f ~/.config/niri/config.kdl
readlink -f ~/.config/kitty/kitty.conf
```

Both links must resolve inside
`~/Projects/CycloniteRDX/niri-dotfiles`.

Niri live-reloads valid configuration. Open a new Kitty with `Super+Enter` and
confirm the new palette. If the current Niri process reports a configuration
error, keep the working session open, fix the tracked file, and rerun
`niri validate`; do not log out while the configuration is invalid.

## Functional test

Open three Kitty windows and exercise every shortcut family:

1. Focus and move left/right between columns.
2. Stack two windows vertically and focus/move up/down.
3. Cycle all three width and height presets.
4. Maximize, fullscreen, center, float, return to tiling, and toggle tabs.
5. Create another dynamic workspace and move a column to it.
6. Reorder workspaces and verify Waybar follows the Niri state.
7. Open the overview and hotkey overlay.
8. Verify launcher, screenshots, volume, microphone, brightness, lock, and exit
   bindings inherited from earlier chapters.
9. Open Firefox picture-in-picture and confirm only that player floats.

Do not continue with a binding that silently does nothing. Check the Niri log
and current action spelling before changing the design.

## Restart and greetd verification

Save work, exit Niri through its confirmation dialog, and log in again through
tuigreet. Confirm every session component starts exactly once:

```bash
pgrep -a niri
pgrep -a waybar
pgrep -a mako
pgrep -a swaybg
pgrep -a swayidle
```

Open Kitty and verify:

```bash
niri validate
git -C ~/Projects/CycloniteRDX/niri-dotfiles status --short --branch
```

The repository must remain clean. Runtime programs must not write generated
state through a Stow symlink into tracked configuration.

## Reproduce from a clean target directory

This test proves the package layout without touching the active home:

```bash
mkdir -m 0700 ~/stow-handoff-test
stow --verbose --no-folding --target="$HOME/stow-handoff-test" niri autostart mimeapps waybar fuzzel mako wallpapers swaylock kitty
find -L ~/stow-handoff-test -type l -print
find ~/stow-handoff-test -type l -printf '%p -> %l\n' | sort
```

`find -L ... -type l` must print nothing; that command reports broken symbolic
links. The second command lists the valid reconstruction map. After reviewing
it, remove the test directory through the graphical file manager so it can be
recovered from Trash if needed.

## Rollback

If Kitty alone is faulty:

```bash
stow --delete --verbose --target="$HOME" kitty
```

If the new Niri behavior is faulty, use Git to inspect the chapter 13 commit
and deliberately revert or repair `niri/.config/niri/config.kdl`, then:

```bash
niri validate --config niri/.config/niri/config.kdl
stow --restow --verbose --no-folding --target="$HOME" niri
```

If login becomes inaccessible, use TTY3 and disable greetd as documented in
chapter 11. Never delete the complete dotfiles clone as a recovery shortcut.

## Completion checklist

- [ ] The portable Niri file validates.
- [ ] Kitty parses the tracked configuration and uses Noto Sans Mono.
- [ ] Every shortcut family passes its functional test.
- [ ] Touchpad, TrackPoint, keyboard and gestures remain usable.
- [ ] Waybar follows dynamic Niri workspaces.
- [ ] Lock, idle and greetd behavior still pass after the handoff.
- [ ] Every session component runs exactly once.
- [ ] A clean target directory can be reconstructed with Stow without broken links.
- [ ] The active repository remains clean after login and normal use.
- [ ] No secret or generated state entered Git.

## Sources

- [Niri configuration overview](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Overview)
- [Niri input configuration](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Input)
- [Niri key bindings](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Key-Bindings)
- [Niri layout](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Layout)
- [Niri outputs](https://github.com/YaLTeR/niri/wiki/Configuration%3A-Outputs)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Kitty configuration](https://sw.kovidgoyal.net/kitty/conf/)

