# 22 — Finish Niri v1 and adapt the internal-panel refresh rate

## Goal

Record the first target ThinkPad's finished Niri v1 configuration as a
reproducible checkpoint. This stage refines the compositor itself rather than
replacing any desktop component:

- ergonomic keyboard, touchpad, and TrackPoint behavior;
- scrollable-tiling geometry, focus, window, workspace, floating, and tabbed
  controls;
- the Midnight Circuit focus, shadow, overview, and recent-window treatment;
- the measured internal-panel mode and fractional scale;
- media keys through Playerctl;
- event-driven switching between 60 Hz and 48 Hz according to the TLP profile;
- reapplication of the selected mode after resume.

Waybar, Fuzzel, Mako, swaybg, swaylock, swayidle, Kitty, greetd, and Plymouth
keep their established roles. This chapter adds no second compositor, power
manager, idle daemon, shell, display manager, system service, root helper, or
polling loop.

## Status and hardware boundary

This chapter passed hardware validation on the first target Lenovo ThinkPad
T14 Gen 1 AMD on 2026-09-07. Its accepted internal display reports:

| Property | Validated value |
| --- | --- |
| Connector | `eDP-1` |
| Native resolution | `1920x1080` |
| Normal refresh | `60.049 Hz` |
| Power-saving refresh | `48.040 Hz` |
| Scale | `1.25` |
| Logical size | `1536x864` |

These are measured host values, not guesses or portable defaults. Do not
deploy this checkpoint blindly on the second ThinkPad, after a panel
replacement, or on a machine whose `niri msg outputs` report differs. The
shared external-display policy remains automatic; the refresh helper changes
only `eDP-1`.

The immutable checkpoint name for the complete result is
`post-install-22-v1` in both `arch-linux-post-install` and
`niri-dotfiles`. Each tag points to its own repository's final chapter 22
commit. The earlier dotfiles `post-install-21-v1` tag deliberately points to
`b922d85`, the last Waybar-only state. Later Niri commits also refine Waybar
speaker scrolling to 2%; that cumulative interaction belongs to chapter 22 and
does not justify rewriting the chapter 21 tag.

## Prerequisites

Before applying this chapter:

- chapters 00 through 21 are complete;
- Waybar is usable and the two icon fonts from chapter 21 are installed;
- TLP and `tlp-pd.service` from chapter 06 are active and remain the only
  hardware power-policy owner;
- the first target's `niri msg outputs` report contains both exact modes;
- the current Niri session, TTY3, and tuigreet recovery path work;
- the dotfiles clone is clean and synchronized;
- important work is saved before the logout/login test.

Run the Arch commands as `neon` from Kitty unless a step says otherwise.
Repository commits, tags, and pushes are prepared separately from Windows.

## Preserve the ownership model

| Concern | Owner after this chapter |
| --- | --- |
| Windows, workspaces, input, layout, output mode, and scale | Niri |
| AC/battery hardware tuning and selected power profile | TLP |
| Standard three-profile D-Bus interface | `tlp-pd` |
| Mapping a profile event to the internal-panel mode | `power-profile-refresh.py` |
| Lock, monitor-off, and idle timing | swayidle |
| Authentication | swaylock plus PAM |
| Status presentation and narrow controls | Waybar |
| Media-player dispatch | Playerctl and each MPRIS-capable application |
| Resume notification | systemd-logind's existing D-Bus interface |

The helper observes `ActiveProfile`; it never selects a TLP profile. It asks
Niri to change one advertised mode and never writes DRM sysfs, runs as root, or
creates a system unit.

## Audit the measured baseline

Confirm the display identity and exact advertised timings before selecting the
candidate:

```bash
hostnamectl
pacman -Q niri tlp tlp-pd
niri --version
niri msg outputs
tlpctl get
systemctl is-active tlp-pd.service
systemctl is-active power-profiles-daemon.service 2>&1
```

The internal report must name `eDP-1` and advertise both
`1920x1080@60.049` and `1920x1080@48.040`. `tlp-pd.service` must be
active. The separate `power-profiles-daemon.service` must remain absent or
inactive.

Stop if a mode string differs even slightly. Niri mode identifiers include the
exact three-decimal refresh rate; `60` is not a safe substitute for
`60.049`.

## Select the reviewed candidate

For the original validation, update the development branch:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch main
git pull --ff-only
git status --short --branch
git log --oneline --decorate -6
```

Stop on local changes or a non-fast-forward update. For a later reconstruction,
select the immutable cumulative checkpoint instead:

```bash
git switch --detach post-install-22-v1
git describe --tags --exact-match
```

Do not merge `post-install-21-v1` into a newer branch. Chapter tags are
cumulative snapshots selected with `git switch --detach`, not migrations.

## Install the explicit runtime dependencies

The final configuration uses two packages that must not remain accidental
transitive dependencies:

| Package | Consumer | Purpose |
| --- | --- | --- |
| `playerctl` | Niri media-key bindings | Dispatch play/pause, stop, previous, and next through MPRIS |
| `python-gobject` | Refresh helper | Provide Python bindings for GLib and GIO, including system D-Bus proxies |

Both are in Arch's official `Extra` repository. Install them through a full
upgrade:

```bash
sudo pacman -Syu playerctl python-gobject
pacman -Q playerctl python-gobject
command -v playerctl python3 gdbus
python3 -c 'from gi.repository import Gio, GLib; print(Gio.Application, GLib.MainLoop)'
```

The import must succeed. No AUR package or Python virtual environment belongs
in this session integration.

## Review the exact tracked change

The functional files involved in this checkpoint are:

```text
niri/.config/niri/config.kdl
niri/.config/niri/scripts/power-profile-refresh.py
waybar/.config/waybar/config.jsonc
```

The Waybar file is included because its final speaker step and scroll
directions were coordinated with Niri's input refinement. Chapter 22 does not
change Waybar's visual role or add a module.

Inspect the files and the cumulative difference:

```bash
sed -n '1,760p' niri/.config/niri/config.kdl
sed -n '1,220p' niri/.config/niri/scripts/power-profile-refresh.py
sed -n '1,260p' waybar/.config/waybar/config.jsonc
git diff --check post-install-21-v1..HEAD
git diff --stat post-install-21-v1..HEAD
git diff post-install-21-v1..HEAD -- niri/.config/niri/config.kdl
git diff post-install-21-v1..HEAD -- niri/.config/niri/scripts/power-profile-refresh.py
git diff post-install-21-v1..HEAD -- waybar/.config/waybar/config.jsonc
```

Validate the formats without executing the helper:

```bash
niri validate --config niri/.config/niri/config.kdl
jq empty waybar/.config/waybar/config.jsonc
python3 -c 'import ast, pathlib; ast.parse(pathlib.Path("niri/.config/niri/scripts/power-profile-refresh.py").read_text())'
git ls-files --stage niri/.config/niri/scripts/power-profile-refresh.py
```

The Git mode in the final command must be `100755`. If it is `100644`, stop
and correct the index in the publishing checkout with:

```bash
git add --chmod=+x niri/.config/niri/scripts/power-profile-refresh.py
```

A local `chmod` that is not recorded by Git is not a reproducible fix,
especially when the repository is prepared through a ZIP on Windows.

## Niri v1 contract

### Keyboard and pointing devices

The selected input policy is:

- US layout;
- right Alt as Compose;
- Caps Lock remapped to Ctrl;
- 300 ms repeat delay and 25 characters per second;
- touchpad tap, disable-while-typing, disable-while-trackpointing, drag, and
  natural scrolling;
- touchpad scroll factor `0.70` and acceleration `0.15`;
- one-, two-, and three-finger taps mapped to left, right, and middle click;
- TrackPoint acceleration `-0.40`;
- mouse defaults unchanged;
- focus follows the pointer and the pointer warps to keyboard focus;
- workspace auto-back-and-forth enabled.

These are preferences validated on this ThinkPad. They do not change the TTY
keymap, tuigreet keymap, or another compositor's input configuration.

### Output and layout

The tracked `eDP-1` block selects the measured 60.049 Hz mode and scale 1.25
at session startup. The layout then uses:

- 10 px gaps;
- centered focused columns when overflowed;
- one-third, one-half, and two-thirds width and height presets;
- half-width default columns;
- a 2 px cyan-to-fuchsia focus ring;
- no separate border;
- a restrained shadow;
- 8 px clipped window corners;
- dark overview backdrop;
- Niri's default animation timings;
- Alt-Tab recent-window previews.

No external connector, position, transform, variable-refresh rule, or EDID
serial is hard-coded.

### Daily bindings

In this table, `Super` is Niri's `Mod` key:

| Shortcut | Action |
| --- | --- |
| `Super+T` or `Super+Enter` | Open Kitty |
| `Super+D` | Open Fuzzel |
| `Super+O` | Toggle overview |
| `Super+Q` | Close the focused window |
| `Super+Shift+L` | Lock with swaylock |
| `Super+Shift+/` | Show the important-shortcut overlay |
| `Super+H/J/K/L` | Move focus left/down/up/right |
| `Super+Alt+H/J/K/L` | Move the window or column |
| `Super+Ctrl+J/K` | Change workspace down/up |
| `Super+Ctrl+Alt+J/K` | Move the column to another workspace |
| `Super+Shift+J/K` | Reorder workspaces |
| `Super+1…9` | Focus a numbered workspace |
| `Super+Alt+1…9` | Move the column to a numbered workspace |
| `Super+R` / `Super+Alt+R` | Cycle width presets forward/backward |
| `Super+Ctrl+R` | Cycle window-height presets |
| `Super+F` | Maximize the column |
| `Super+Alt+F` | Toggle true fullscreen |
| `Super+M` | Maximize the window to the output edges |
| `Super+Ctrl+F` | Expand into available width |
| `Super+V` | Toggle floating |
| `Super+Alt+V` | Switch focus between tiling and floating |
| `Super+W` | Toggle tabbed display for the column |
| `Print`, `Ctrl+Print`, `Alt+Print` | Interactive, output, or window screenshot |
| Media and volume hardware keys | Control MPRIS playback and PipeWire endpoints |
| Brightness hardware keys | Adjust the kernel backlight |
| `Super+Shift+E` | Exit Niri through its confirmation dialog |

The full tracked configuration remains the authority for less frequent sizing,
centering, consume/expel, and mouse-wheel bindings.

## Understand the adaptive-refresh helper

The helper is started once by Niri:

```kdl
spawn-sh-at-startup "$HOME/.config/niri/scripts/power-profile-refresh.py"
```

It creates two system-bus observers:

1. `org.freedesktop.UPower.PowerProfiles`, implemented here by `tlp-pd`,
   for `ActiveProfile` changes;
2. `org.freedesktop.login1.Manager` for `PrepareForSleep`, so the mode is
   reapplied one second after resume.

The mapping is:

| TLP profile | Requested Niri mode |
| --- | --- |
| `performance` | `eDP-1 1920x1080@60.049` |
| `balanced` | `eDP-1 1920x1080@60.049` |
| `power-saver` | `eDP-1 1920x1080@48.040` |

The long-lived GLib main loop sleeps until D-Bus emits an event. It does not
poll and does not infer AC/battery state itself. TLP remains free to choose its
normal `performance` profile on AC and `balanced` profile on battery; both
keep 60 Hz. The panel changes to 48 Hz only when the explicit
`power-saver` profile is active.

## Preview and deploy

Preview both affected Stow packages:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --restow --simulate --verbose --no-folding --target="$HOME" niri waybar
```

The preview must mention only tracked targets from those two packages. Deploy:

```bash
stow --restow --verbose --no-folding --target="$HOME" niri waybar
readlink -f ~/.config/niri/config.kdl
readlink -f ~/.config/niri/scripts/power-profile-refresh.py
readlink -f ~/.config/waybar/config.jsonc
test -x ~/.config/niri/scripts/power-profile-refresh.py
niri validate
jq empty ~/.config/waybar/config.jsonc
```

Startup commands are evaluated when a new Niri session starts. Save work, use
`Super+Shift+E`, return through the unchanged tuigreet, and log in again. Do
not launch a second helper manually alongside the session-owned instance.

## Validate the finished session

### Process and logs

```bash
pgrep -af '[p]ower-profile-refresh.py'
watcher_count="$(pgrep -fc '[p]ower-profile-refresh.py' || true)"
printf 'Refresh watchers: %s\n' "$watcher_count"
test "$watcher_count" -eq 1
niri validate
systemctl --user --failed --no-pager
journalctl --user -b --since '10 minutes ago' --no-pager |
  grep -Ei 'niri|power-profile-refresh|waybar|error|warning|failed' || true
```

Exactly one helper must exist. A warning needs interpretation; an invalid Niri
file, repeated modeset failure, missing Python module, or second watcher fails
the checkpoint.

### Input, layout, and bindings

Validate on real hardware:

1. type ordinary text, use right-Alt Compose, and confirm Caps Lock behaves as
   Ctrl;
2. test touchpad tap, natural scroll, drag, palm rejection, and TrackPoint
   movement;
3. compare mouse and touchpad scrolling in Waybar: both directions must feel
   intentional;
4. create several windows and workspaces and exercise focus, move, size,
   floating, tabs, overview, and recent-window switching;
5. confirm volume changes in 2% steps from the speaker module and hardware
   keys, while microphone and brightness scrolling remain at 5%;
6. test media, screenshot, brightness, lock, and confirmed exit bindings;
7. verify that Waybar, Mako, swaybg, swayidle, and the polkit agent still start
   exactly once.

Useful comparisons:

```bash
niri msg workspaces
wpctl get-volume @DEFAULT_AUDIO_SINK@
wpctl get-volume @DEFAULT_AUDIO_SOURCE@
brightnessctl info
playerctl status
pgrep -a waybar
pgrep -a mako
pgrep -a swaybg
pgrep -a swayidle
```

`playerctl status` may report that no players were found when no
MPRIS-compatible application is open; that is not a dependency failure.

### Profile-to-refresh mapping

Read the D-Bus property and current output before changing anything:

```bash
tlpctl get
gdbus call --system --dest org.freedesktop.UPower.PowerProfiles \
  --object-path /org/freedesktop/UPower/PowerProfiles \
  --method org.freedesktop.DBus.Properties.Get \
  org.freedesktop.UPower.PowerProfiles ActiveProfile
niri msg outputs
```

Exercise all three explicit profiles, waiting briefly for the D-Bus event after
each command:

```bash
tlpctl power-saver
sleep 2
niri msg outputs

tlpctl balanced
sleep 2
niri msg outputs

tlpctl performance
sleep 2
niri msg outputs
```

The observed sequence must be 48.040, 60.049, and 60.049 Hz. Waybar's
power-profile icon and tooltip must follow the same three states. An external
display, if connected, must not be modeset by the helper.

Return TLP to its source-based automatic choice after the test:

```bash
sudo tlp start
tlpctl get
niri msg outputs
```

Normally that means `performance` on AC and `balanced` on battery, both at
60.049 Hz.

### Suspend and resume

Save work, suspend once, and confirm that swaylock protects the resumed
session. After unlock:

```bash
test "$(pgrep -fc '[p]ower-profile-refresh.py' || true)" -eq 1
tlpctl get
niri msg outputs
systemctl --user --failed --no-pager
```

The mode must agree with the active profile. The helper must survive the
suspend in the same Niri session and reapply the correct mode after resume.

## Roll back to the Waybar-only checkpoint

If Niri cannot validate, input becomes unusable, the internal display modesets
incorrectly, or the watcher duplicates or fails repeatedly, use TTY3. Save any
useful logs, then remove the current Niri links while its complete package
still exists:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
stow --delete --verbose --target="$HOME" niri
stow --delete --verbose --target="$HOME" waybar
git switch --detach post-install-21-v1
git describe --tags --exact-match
stow --restow --verbose --no-folding --target="$HOME" niri waybar
niri validate
jq empty ~/.config/waybar/config.jsonc
```

Then end the affected Niri session and start a fresh one. This rollback removes
the tracked refresh helper link and returns both Niri and the coordinated
Waybar input settings to the chapter 21 state. It does not change TLP,
`tlp-pd`, logind, swayidle, system services, or boot files. The two official
runtime packages may remain installed.

To return to development after diagnosis:

```bash
git switch main
git pull --ff-only
```

## Completion checklist

- [ ] The internal connector and both exact refresh modes match the measured first target.
- [ ] `playerctl` and `python-gobject` are explicitly installed.
- [ ] Niri, JSON, and Python syntax validation pass.
- [ ] Git records the helper as `100755`, and the deployed link is executable.
- [ ] Stow preview and deployment affect only the intended packages.
- [ ] A fresh tuigreet login starts exactly one refresh helper.
- [ ] Keyboard, touchpad, TrackPoint, focus-follow-pointer, and pointer warp behave as selected.
- [ ] Window, workspace, sizing, floating, tabbed, overview, and recent-window controls work.
- [ ] Media, audio, brightness, screenshot, lock, and exit bindings work.
- [ ] Waybar speaker control and Niri volume keys use 2% steps.
- [ ] TLP remains the sole power-policy owner and `tlp-pd` exposes its profile.
- [ ] Power-saver selects 48.040 Hz; balanced and performance select 60.049 Hz.
- [ ] TLP returns to automatic source-based selection after testing.
- [ ] Suspend resumes behind swaylock with one helper and the correct mode.
- [ ] External outputs remain outside the helper's target.
- [ ] TTY3 and the chapter 21 rollback remain usable.
- [ ] The second ThinkPad is not assumed to share this output policy.
- [ ] `post-install-22-v1` is created at the final hardware-validated documentation commit in both post-install and dotfiles.

This complete configuration passed the input, layout, profile-change, and
suspend/resume validation on the first target on 2026-09-07.

## Sources

- [Niri: input configuration](https://niri-wm.github.io/niri/Configuration:-Input.html)
- [Niri: output configuration](https://niri-wm.github.io/niri/Configuration:-Outputs.html)
- [Niri: key bindings](https://niri-wm.github.io/niri/Configuration:-Key-Bindings.html)
- [Niri: recent windows](https://niri-wm.github.io/niri/Configuration:-Recent-Windows.html)
- [Niri IPC](https://niri-wm.github.io/niri/IPC.html)
- [Arch package: playerctl](https://archlinux.org/packages/extra/x86_64/playerctl/)
- [Arch package: python-gobject](https://archlinux.org/packages/extra/x86_64/python-gobject/)
- [TLP: power-profiles-daemon compatibility](https://linrunner.de/tlp/faq/ppd.html)
- [Gio.DBusProxy](https://docs.gtk.org/gio/class.DBusProxy.html)
- [systemd logind D-Bus interface](https://www.freedesktop.org/software/systemd/man/latest/org.freedesktop.login1.html)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next step

Personalize Fuzzel while keeping the validated Waybar and Niri checkpoints
unchanged. The launcher chapter should change one surface, preserve application
discovery, and prove keyboard, mouse, scaling, theme, and rollback behavior
before Mako is restyled.
