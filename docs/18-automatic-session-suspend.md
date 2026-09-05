# 18 — Add battery-only automatic session suspend

## Goal

Extend the proven lock and monitor-power lifecycle with one conservative final
stage:

| Idle time or event | Result |
| --- | --- |
| 5 minutes | swaylock covers the Niri session. |
| 10 minutes | Niri powers the monitors off. |
| User activity | Niri powers the monitors on and resets the progression. |
| 30 minutes on battery | Request system suspend. |
| 30 minutes on external power | Stay awake. |
| Any coordinated sleep | swaylock is ready before sleep continues. |

swayidle remains the only session-idle coordinator. A small dotfiles helper
reads UPower's `OnBattery` property and requests suspend through systemd with
inhibitor checking explicitly enabled. It fails closed: unknown power state,
external power, an active sleep inhibitor, or a refused authorization leaves
the machine awake.

This chapter does not configure hibernation, suspend on AC, post-logout idle
policy, `IdleAction=` in logind, a second idle daemon, a systemd timer, a TLP
profile, or a passwordless privilege rule.

## Status and prerequisites

This chapter is reviewed and awaits hardware validation.

Before applying it:

- chapters 00 through 17 are complete;
- chapter 17 is hardware-validated;
- manual and lid-triggered suspend still resume behind swaylock;
- UPower, swayidle, swaylock, and systemd are installed and healthy;
- exactly one swayidle process runs in the Niri session;
- the current dotfiles deployment and clone are clean;
- TTY3 remains available for recovery.

Run the procedure as `neon` from Kitty. Save work before every real suspend
test.

## Select the chapter 18 checkpoint

The dotfiles change must already be committed and tagged. Select it exactly:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch --detach post-install-18-v1
git describe --tags --exact-match
git log -1 --oneline
```

Stop on a local change, a missing tag, or a different exact description.
Detached HEAD is deliberate for a reproducible installation checkpoint.

The tag reference and its annotation are different fields. The command used
when publishing this checkpoint is:

```bash
git tag -a post-install-18-v1 \
  -m "Post-install chapter 18 automatic session suspend"
```

`post-install-18-v1` is the name used by `git switch`; the quoted text is the
human-readable message. Verify both with:

```bash
git tag --list 'post-install-18-v1'
git tag -n1 'post-install-18-v1'
```

## Preserve the ownership model

The new timeout changes one policy without changing the established owners:

| Concern | Owner |
| --- | --- |
| Session inactivity and timeout ordering | One swayidle process started by Niri |
| Lock surface and authentication | swaylock plus PAM |
| Monitor power | Niri IPC actions |
| AC versus battery observation | UPower system service |
| Suspend request and inhibitors | systemd-logind/systemd |
| Hardware sleep transition | systemd sleep units, kernel, and firmware |
| Laptop tuning and charge thresholds | TLP |

Do not also set `IdleAction=suspend`, install another idle daemon, or create a
user timer. Those would introduce a second clock that cannot share swayidle's
Wayland activity state.

The final timeout does not call `sudo`. An active local graphical session may
request suspend through logind's normal policy. If that policy refuses the
request, the helper records the refusal and stays awake.

## Audit the current baseline

Confirm the installed components and system state:

```bash
pacman -Q swayidle swaylock upower systemd
systemctl is-active upower.service
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

UPower may be D-Bus activated; if the first status is `inactive`, query its
property before deciding that it is broken:

```bash
busctl --system get-property \
  org.freedesktop.UPower \
  /org/freedesktop/UPower \
  org.freedesktop.UPower \
  OnBattery
systemctl is-active upower.service
```

The property returns `b false` on external power and `b true` on battery.
Anything else is an unresolved boundary; do not infer the source from the
battery percentage.

Confirm there is exactly one existing idle coordinator and record its complete
command line:

```bash
swayidle_count=$(pgrep -xc swayidle || true)
printf 'swayidle processes: %s\n' "$swayidle_count"
test "$swayidle_count" -eq 1
swayidle_pid=$(pgrep -xo swayidle)
printf 'swayidle PID: %s\n' "$swayidle_pid"
tr '\0' ' ' <"/proc/$swayidle_pid/cmdline"
printf '\n'
systemd-inhibit --list
```

Stop if `pgrep` finds none or more than one process. The current command must
contain the 300- and 600-second stages and the `before-sleep` locker, but no
automatic suspend stage yet.

Check common competing policy locations:

```bash
systemd-analyze cat-config systemd/logind.conf
grep -RnsE '^[[:space:]]*Idle(Action|ActionSec)=' \
  /etc/systemd /run/systemd /usr/local/lib/systemd /usr/lib/systemd \
  2>/dev/null || true
pgrep -a swayidle
pgrep -a hypridle
```

The canonical merged logind policy leaves `IdleAction=ignore`; there is no
hypridle process and no second swayidle process. Do not remove an unexpected
owner until its origin is understood.

## Review the dotfiles change

Chapter 18 changes only:

```text
niri/.config/niri/config.kdl
scripts/.local/bin/idle-suspend
```

The Niri startup line retains the previous lifecycle and adds:

```text
timeout 1800 $HOME/.local/bin/idle-suspend
```

swayidle executes timeout commands through a shell, so `$HOME` expands for the
running `neon` session. The helper itself:

1. queries the boolean UPower system-bus property;
2. exits successfully without sleeping on AC or unknown state;
3. offers a dry-run path for safe validation;
4. calls `systemctl --check-inhibitors=yes suspend` only on battery;
5. logs every decision with the identifier `idle-suspend`;
6. treats a refused automatic request as a safe skip.

Inspect and validate both files before deployment:

```bash
sed -n '50,85p' niri/.config/niri/config.kdl
sed -n '1,220p' scripts/.local/bin/idle-suspend
test -x scripts/.local/bin/idle-suspend
sh -n scripts/.local/bin/idle-suspend
niri validate --config niri/.config/niri/config.kdl
```

No package installation or system file modification is required. `busctl`,
`systemctl`, and `systemd-cat` come from the already installed systemd package;
UPower was installed in chapter 06.

## Preview and deploy with Stow

The `scripts` package is new. Confirm the live target does not already exist:

```bash
test ! -e ~/.local/bin/idle-suspend
```

No output and exit status zero are expected. If a target exists, inspect it and
back it up outside the active path. Do not use `stow --adopt`, because that can
rewrite the reviewed repository file.

Preview both parts:

```bash
stow --simulate --verbose --no-folding --target="$HOME" scripts
stow --restow --simulate --verbose --no-folding --target="$HOME" niri
```

When the preview is clean:

```bash
stow --verbose --no-folding --target="$HOME" scripts
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
readlink -f ~/.local/bin/idle-suspend
readlink -f ~/.config/niri/config.kdl
test -x ~/.local/bin/idle-suspend
sh -n ~/.local/bin/idle-suspend
```

Both links must resolve inside the selected `niri-dotfiles` checkout.

## Validate power detection without sleeping

First connect external power and confirm:

```bash
busctl --system get-property \
  org.freedesktop.UPower \
  /org/freedesktop/UPower \
  org.freedesktop.UPower \
  OnBattery
IDLE_SUSPEND_DRY_RUN=1 ~/.local/bin/idle-suspend
```

Expected results are `b false` and a message saying that external power is
connected. The dry-run variable is irrelevant here because AC already causes a
safe skip.

Unplug the charger, wait several seconds, and repeat:

```bash
busctl --system get-property \
  org.freedesktop.UPower \
  /org/freedesktop/UPower \
  org.freedesktop.UPower \
  OnBattery
IDLE_SUSPEND_DRY_RUN=1 ~/.local/bin/idle-suspend
```

Expected results are `b true` and `Dry run: battery detected; suspend would be
requested.` The machine must remain awake. Reconnect AC when the test is done.

Inspect the helper's journal records:

```bash
journalctl -t idle-suspend -n 20 --no-pager
```

If journal permissions hide the records, the same messages already appeared
on the invoking terminal; do not weaken journal permissions for this test.

## Prove that sleep inhibitors are honored

This test must run on battery so that the helper reaches its suspend request.
Save work first.

In one Kitty window, hold a temporary blocking sleep inhibitor:

```bash
systemd-inhibit \
  --what=sleep \
  --mode=block \
  --who="Chapter 18 validation" \
  --why="Prove automatic suspend respects blockers" \
  sleep 120
```

In a second Kitty, confirm the inhibitor and call the real helper:

```bash
systemd-inhibit --list --what=sleep
~/.local/bin/idle-suspend
```

The suspend request must be refused and the desktop must remain awake. The
helper exits successfully after recording the refusal; it never retries by
bypassing the inhibitor. Stop the temporary `sleep` with `Ctrl+C` in the first
window and reconnect AC.

An application-level Wayland idle inhibitor and a systemd sleep inhibitor are
different boundaries. Video or presentation software should prevent the
compositor idle sequence when it implements the Wayland protocol. A long build
or download may instead be protected explicitly:

```bash
systemd-inhibit \
  --what=sleep \
  --mode=block \
  --why="Local build must finish" \
  make -j"$(nproc)"
```

That wrapper protects the final sleep request; it does not promise to keep the
screen unlocked or powered on. Perform unattended upgrades on AC, where this
chapter never auto-suspends.

## Start a fresh Niri session

Niri live-reloads configuration, but `spawn-at-startup` is not a restart
manager for an already running swayidle process. Save work, exit Niri with
`Super+Shift+E`, and log in again through tuigreet.

Confirm exactly one new process contains all three timeouts:

```bash
swayidle_count=$(pgrep -xc swayidle || true)
printf 'swayidle processes: %s\n' "$swayidle_count"
test "$swayidle_count" -eq 1
swayidle_pid=$(pgrep -xo swayidle)
printf 'swayidle PID: %s\n' "$swayidle_pid"
tr '\0' ' ' <"/proc/$swayidle_pid/cmdline"
printf '\n'
```

The command line must show `timeout 300`, `timeout 600`, `timeout 1800`, the
helper path, and `before-sleep swaylock -f`. If it still shows only the old two
timeouts, the session was not restarted or a different startup owner exists.

## Exercise the complete progression without waiting 30 minutes

swayidle handles `SIGUSR1` by entering its idle state immediately. This is a
supervised integration test of the committed timeouts; it is not a daily
shortcut.

### Test on external power

Connect AC, save work, keep TTY3 available, then run:

```bash
swayidle_pid=$(pgrep -xo swayidle)
kill -USR1 "$swayidle_pid"
```

The session should lock, the displays should power off, and the helper should
skip suspend because UPower reports external power. Input must power the
displays on and reveal swaylock, never the desktop. Unlock normally and check:

```bash
journalctl -t idle-suspend -n 10 --no-pager
```

### Test on battery

Unplug AC, confirm `b true`, save all work, and repeat the signal:

```bash
busctl --system get-property \
  org.freedesktop.UPower \
  /org/freedesktop/UPower \
  org.freedesktop.UPower \
  OnBattery
swayidle_pid=$(pgrep -xo swayidle)
kill -USR1 "$swayidle_pid"
```

The expected path is lock, monitors off, suspend, wake, and swaylock before any
desktop content. A brief desktop flash is a failure even when suspend itself
works.

After unlocking, reconnect AC and inspect:

```bash
journalctl -b -u systemd-suspend.service --no-pager
journalctl -b -t idle-suspend --no-pager
journalctl -b --no-pager | grep -Ei 'suspend|sleep|swayidle|swaylock'
nmcli general status
bluetoothctl show
wpctl status
sudo tlp-stat -s
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

Wi-Fi, Bluetooth, audio, input, TLP, monitor wake, and the user/system unit
states must remain healthy.

## Observe real timers and application inhibitors

The signal test proves the event chain but not ordinary elapsed-time behavior.
Under supervision, verify the real policy at least once:

1. on AC, leave the session untouched through 30 minutes; it must lock and
   power monitors off but stay awake;
2. on battery, leave it untouched through 30 minutes; it must suspend and wake
   behind swaylock;
3. during video or a presentation known to request Wayland idle inhibition,
   confirm the five- and ten-minute actions do not interrupt playback;
4. while a blocking sleep inhibitor is visible in `systemd-inhibit --list`,
   confirm the final suspend request is refused;
5. after genuine input, confirm the entire progression starts over.

If a systemd-only inhibitor disappears after the 30-minute request was
refused, this conservative implementation does not schedule an independent
retry while the session remains untouched. New user activity resets swayidle,
and the next complete idle cycle may try again. This avoids a second timer that
could suspend after the user had become active.

## Rollback

If automatic suspend is too aggressive or any resume test fails, remove only
the new stage while preserving the proven lock and monitor lifecycle.

From TTY3 or a usable Niri session:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --delete --verbose --target="$HOME" scripts
git switch --detach post-install-17-v1
niri validate --config niri/.config/niri/config.kdl
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
```

Save work, exit Niri, and log in again. Confirm the new swayidle command line no
longer includes `timeout 1800`. The five-minute lock, ten-minute display-off,
activity resume, manual suspend, and lid suspend paths must still work.

Do not disable systemd-logind, remove swaylock, alter PAM, or force inhibitor
bypass to solve an automatic-timer problem.

## Completion checklist

- [ ] Chapter 17 is recorded as hardware-validated.
- [ ] `post-install-18-v1` is the selected exact dotfiles tag.
- [ ] swayidle, swaylock, UPower, and systemd packages are installed and healthy.
- [ ] One swayidle process owns the 300-, 600-, and 1800-second progression.
- [ ] Merged logind policy has no competing automatic idle action.
- [ ] The tracked helper is executable, passes `sh -n`, and is Stow-managed.
- [ ] UPower reports `b false` on AC and `b true` on battery.
- [ ] Dry run skips on AC and reaches the battery decision without sleeping.
- [ ] A blocking sleep inhibitor prevents the helper from suspending.
- [ ] Forced idle on AC locks and powers displays off without suspending.
- [ ] Forced idle on battery suspends and resumes behind swaylock.
- [ ] Real 30-minute timers behave correctly on AC and battery.
- [ ] A tested Wayland-aware media or presentation application inhibits idle.
- [ ] Wi-Fi, Bluetooth, audio, TLP, input, outputs, and units recover cleanly.
- [ ] Hibernation, automatic suspend on AC, and post-logout suspend remain absent.
- [ ] The dotfiles repository remains clean after normal operation.

## Sources

- [swayidle manual](https://man.archlinux.org/man/swayidle.1)
- [swaylock manual](https://man.archlinux.org/man/swaylock.1)
- [UPower D-Bus interface](https://upower.freedesktop.org/docs/UPower.html)
- [systemd inhibitor locks](https://systemd.io/INHIBITOR_LOCKS/)
- [systemd-inhibit manual](https://man.archlinux.org/man/systemd-inhibit.1)
- [systemctl manual](https://man.archlinux.org/man/systemctl.1)
- [logind.conf manual](https://man.archlinux.org/man/logind.conf.5)
- [Niri configuration: miscellaneous](https://niri-wm.github.io/niri/Configuration:-Miscellaneous.html)
- [ArchWiki: Power management](https://wiki.archlinux.org/title/Power_management)
- [ArchWiki: Suspend and hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate)
