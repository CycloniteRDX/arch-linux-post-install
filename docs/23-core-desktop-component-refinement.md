# 23 — Refine the core desktop components

## Goal

Record the first coordinated refinement of the remaining everyday desktop
surfaces after Waybar and Niri reached their own stable checkpoints:

- make Fuzzel compact, predictable, and useful both as a launcher and picker;
- add clear urgency, history, progress, icons, and actions to Mako;
- refine swaylock without changing PAM or the lock lifecycle;
- make Kitty visually coherent and improve its daily interaction defaults;
- ensure monitors are powered on when logind reports a system resume.

This chapter does not replace a component or change its ownership. It adds no
daemon, system service, root helper, AUR package, or new power-policy provider.

## Status

The five dotfiles commits covered by this chapter passed hardware validation on
the first target Lenovo ThinkPad T14 Gen 1 AMD on 2026-09-08:

| Commit | Area | Result |
| --- | --- | --- |
| `9115442` | Kitty | Cursor trails, command notifications, scrolling, tabs, safety, and appearance validated |
| `c0b5b82` | Mako | Presentation, urgency, history, progress, and pointer actions validated |
| `3ae8ac4` | Fuzzel | Launcher layout, matching, counter, overlay, keyboard, and pointer use validated |
| `eca9a56` | swaylock | Indicator states, bad password, successful unlock, and styling validated |
| `0e7af47` | swayidle/Niri | Monitors restored correctly after system resume |

The immutable cumulative checkpoint is `post-install-23-v1` in both
`arch-linux-post-install` and `niri-dotfiles`. Create each tag only after its
final chapter 23 documentation commit; the same tag name intentionally points
to a different repository-specific commit.

## Prerequisites

Before reconstructing or revalidating this stage:

- chapters 00 through 22 are complete;
- `post-install-22-v1` remains available as the known-good rollback;
- Kitty, Fuzzel, Mako, swaylock, swayidle, libnotify, and GNU Stow are installed;
- the Niri session, TTY3, and tuigreet recovery paths work;
- the dotfiles clone is clean and synchronized;
- important work is saved before lock and suspend tests.

No new package is introduced. Mako's middle-click action selector deliberately
reuses Fuzzel, which already belongs to the selected desktop stack.

## Preserve component ownership

| Concern | Owner after this chapter |
| --- | --- |
| Application discovery and general-purpose selection | Fuzzel |
| `org.freedesktop.Notifications`, visible notifications, and history | Mako |
| Lock surface | swaylock |
| Unlock authentication | swaylock through its existing PAM service |
| Idle, monitor-off, pre-sleep, and post-resume coordination | swayidle |
| Windows, outputs, and monitor power actions | Niri |
| Terminal rendering and terminal-local interaction | Kitty |
| Hardware power profile | TLP plus `tlp-pd` |

Mako remains the only notification daemon. swaylock does not replace greetd,
and swayidle does not authenticate. The new `after-resume` event asks the
already-running Niri session to power on its monitors; it does not change TLP,
logind, or the profile-to-refresh helper.

## Select the reviewed state

For a normal reconstruction after the tag exists:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch --detach post-install-23-v1
git describe --tags --exact-match
git log -6 --oneline
```

For active development before the tag is published, use a clean, fast-forwarded
`main` instead. Never merge an older chapter tag into the development branch.

## Review the tracked change

The functional files are:

```text
kitty/.config/kitty/kitty.conf
mako/.config/mako/config
fuzzel/.config/fuzzel/fuzzel.ini
swaylock/.config/swaylock/config
niri/.config/niri/config.kdl
```

Review the cumulative change from the previous immutable checkpoint:

```bash
git diff --check post-install-22-v1..HEAD
git diff --stat post-install-22-v1..HEAD
git diff post-install-22-v1..HEAD -- \
  kitty/.config/kitty/kitty.conf \
  mako/.config/mako/config \
  fuzzel/.config/fuzzel/fuzzel.ini \
  swaylock/.config/swaylock/config \
  niri/.config/niri/config.kdl
```

The expected functional contract is:

| Component | Selected behavior |
| --- | --- |
| Fuzzel | 45-character width, 12 results, fzf matching across name/generic/exec/keywords, match counter, overlay layer, and Midnight Circuit states |
| Mako | Top-right overlay, five visible and twenty historical notifications, 7-second normal timeout, 4-second low timeout, persistent critical notifications, progress overlay, and pointer/touch actions |
| swaylock | Dark background, 60-pixel indicator, 5-pixel ring, transparent separators, cyan verification, amber clear, red error, and failed-attempt count |
| Kitty | Noto Sans Mono 11, 94% background opacity, per-pixel touchpad scrollback, contextual tab bar, clipboard-read confirmation, safer paste, command notification after 15 unfocused seconds, and restrained cursor trail |
| swayidle | Existing activity-resume monitor action plus an explicit logind `after-resume` monitor action |

Kitty's `unfocused` notification policy means that its OS window lacks keyboard
focus. It does not mean that the terminal must be invisible.

## Understand the two resume paths

The swayidle command now contains two similar-looking events with different
triggers:

| Event | Trigger | Purpose |
| --- | --- | --- |
| `timeout 600 ... resume ...` | User activity returns after the ten-minute idle monitor-off timeout | Restore displays blanked by that timeout |
| `after-resume ...` | logind reports that the whole system resumed from sleep | Restore displays after suspend even when the idle timeout's resume edge is not sufficient |

Keeping both is intentional. Neither event starts a second swayidle process,
selects a TLP profile, or changes the internal-panel refresh policy.

## Preview and deploy only the changed packages

From the dotfiles repository root:

```bash
stow --simulate --verbose --no-folding --target="$HOME" \
  kitty mako fuzzel swaylock niri
```

Stop on any conflict. If the preview is correct:

```bash
stow --restow --verbose --no-folding --target="$HOME" \
  kitty mako fuzzel swaylock niri
```

Confirm ownership and baseline syntax:

```bash
readlink -f ~/.config/kitty/kitty.conf
readlink -f ~/.config/mako/config
readlink -f ~/.config/fuzzel/fuzzel.ini
readlink -f ~/.config/swaylock/config
readlink -f ~/.config/niri/config.kdl
niri validate
fuzzel --check-config
```

Reload Mako and inspect the running session:

```bash
makoctl reload
systemctl --user --failed --no-pager
pgrep -a -x mako
pgrep -a -x swayidle
busctl --user status org.freedesktop.Notifications
```

Exactly one Mako and one swayidle process should exist. Fuzzel and swaylock are
normally absent until opened.

## Validate Fuzzel

Open the launcher with `Super+D` and verify:

- it appears above a fullscreen window;
- keyboard typing, arrows, Enter, and Escape work;
- pointer selection and scrolling work;
- the counter updates;
- searches can match an application name, generic name, executable, or keyword;
- launching a terminal application opens it in Kitty;
- text, icons, selection, and matched characters remain legible at scale 1.25.

Also exercise the dmenu path used by Mako:

```bash
printf '%s\n' Alpha Beta Gamma | fuzzel --dmenu --prompt 'Test: '
```

Canceling the picker is valid and must not leave a Fuzzel process behind.

## Validate Mako

Send ordinary, low, and critical notifications:

```bash
notify-send 'Mako normal' 'Seven-second reviewed notification'
notify-send --urgency=low 'Mako low' 'Short low-priority notification'
notify-send --urgency=critical 'Mako critical' 'Dismiss this manually'
notify-send --hint=int:value:65 'Mako progress' '65 percent'
```

Verify normal and low timeouts, persistent critical presentation, progress
rendering, icons, stacking, and fullscreen visibility. Then test:

- left click invokes the default action when one is supplied;
- middle click opens the action list in Fuzzel;
- right click dismisses;
- touch dismisses on a touch-capable target;
- expired notifications appear in `makoctl history`;
- `makoctl restore` returns the newest historical notification.

The chapter 23 checkpoint explicitly searched Adwaita. Final global testing
later proved that the installed Adwaita symbolic layout did not resolve Mako's
tested named icons, while Papirus did. Chapter 27 therefore changes the current
configuration to `/usr/share/icons/Papirus-Dark:/usr/share/icons/Papirus`;
hicolor and pixmaps remain Mako's built-in fallbacks. Keep the historical
chapter 23 tag unchanged.

## Validate swaylock

Save work and run:

```bash
swaylock -f
```

Verify the dark background and compact indicator, enter one deliberately wrong
password, and then unlock with the correct password. The error and verification
states must be readable, and PAM behavior must be unchanged.

Repeat through `Super+Shift+L` and once through suspend. Test every connected
output used by the workstation. A visual refinement is not complete if the
locker exposes an output, accepts an empty password, or returns before the lock
surface is ready.

## Validate Kitty

Start a fresh Kitty window so options that apply only at creation are active.
Verify:

- text, selection, URL hover, cursor, and ANSI colours remain legible;
- the wallpaper is only subtly visible through the 94% background;
- touchpad scrollback is smooth and the scrollbar appears only after scrolling;
- `Ctrl+Shift+Enter` opens a Kitty window/pane in the current directory;
- `Ctrl+Shift+T` opens a tab in the current directory;
- the tab bar appears at two tabs and disappears again at one;
- large cursor jumps show a short magenta trail while normal typing does not;
- clipboard reads still request confirmation and dangerous paste content asks
  before insertion;
- a command lasting at least 15 seconds produces a Mako notification only when
  the Kitty window is unfocused.

The command notification requires Kitty shell integration. If it does not fire,
check that boundary before changing Mako.

## Validate suspend and resume

Confirm one idle coordinator, save work, and suspend:

```bash
pgrep -a -x swayidle
systemctl suspend
```

After resume, the monitors must power on and swaylock must protect the session.
After unlocking:

```bash
pgrep -a -x swayidle
test "$(pgrep -fc '[p]ower-profile-refresh.py' || true)" -eq 1
tlpctl get
niri msg outputs
systemctl --user --failed --no-pager
```

There must still be one swayidle and one refresh helper. The internal refresh
rate must match chapter 22's active-profile mapping, and external outputs must
remain outside that helper's target.

## Roll back to chapter 22

If any surface or resume path fails, use TTY3 if necessary:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
stow --delete --verbose --target="$HOME" kitty mako fuzzel swaylock niri
git switch --detach post-install-22-v1
git describe --tags --exact-match
stow --restow --verbose --no-folding --target="$HOME" \
  kitty mako fuzzel swaylock niri
niri validate
fuzzel --check-config
```

End the affected graphical session and start a fresh one. This rollback changes
only the tracked user configuration; it does not remove packages, edit PAM,
change TLP, or modify system and boot files.

## Completion checklist

- [ ] The cumulative diff contains only the five intended component files.
- [ ] `git diff --check`, `niri validate`, and `fuzzel --check-config` pass.
- [ ] Stow preview and restow report only reviewed targets.
- [ ] Fuzzel launcher and dmenu modes work with keyboard, pointer, icons, scale, and fullscreen windows.
- [ ] Mako normal, low, critical, progress, action, dismiss, history, and restore paths work.
- [ ] Exactly one Mako process owns the notification service.
- [ ] swaylock rejects a bad password, accepts the correct password, and covers every output.
- [ ] Kitty scrolling, tabs, shortcuts, cursor trail, transparency, clipboard/paste safety, and command notification work.
- [ ] Exactly one swayidle process remains after a fresh login and after suspend.
- [ ] Idle monitor restoration and system-resume monitor restoration both work.
- [ ] Resume returns behind swaylock with the refresh helper and TLP mapping intact.
- [ ] TTY3 and the chapter 22 rollback remain usable.
- [ ] `post-install-23-v1` is created at the final documentation commit in both repositories.

This complete set passed real-session, notification, authentication, terminal,
and suspend/resume validation on the first target on 2026-09-08.

## Sources

- [Fuzzel manual](https://man.archlinux.org/man/fuzzel.1.en)
- [Fuzzel configuration manual](https://man.archlinux.org/man/fuzzel.ini.5.en)
- [Mako configuration manual](https://man.archlinux.org/man/mako.5.en)
- [makoctl manual](https://man.archlinux.org/man/makoctl.1.en)
- [Kitty configuration](https://sw.kovidgoyal.net/kitty/conf/)
- [swaylock upstream](https://github.com/swaywm/swaylock)
- [swayidle manual](https://man.archlinux.org/man/swayidle.1.en)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next step

The existing swaybg wallpaper presentation was subsequently accepted as the
finished static policy: one renderer, one selected wallpaper, one fallback
colour, and no rotation or automation scripts. Chapter 24 records tuigreet,
chapter 25 completes the terminal workflow, chapter 26 completes Plymouth, and
chapter 27 records the finished GTK/Qt and cross-component validation.
