# 11 — Configure login, locking, and idle lifecycle

## Goal

Protect the graphical session before making it start automatically. This
chapter adds, in a deliberately safe order:

1. swaylock as the Wayland screen locker;
2. swayidle for idle and pre-suspend coordination;
3. greetd with tuigreet only after locking and suspend recovery are proven.

The resulting lifecycle is:

- `Super+Alt+L` locks immediately;
- five idle minutes lock the session;
- ten idle minutes turn displays off, without suspending the computer;
- user activity turns displays back on;
- every suspend request locks the session before sleep;
- closing the lid continues to follow the logind policy from chapter 06;
- greetd presents a console login and starts `niri-session` after successful
  authentication.

There is no autologin, hibernation, automatic idle suspend, passwordless power
command, or graphical greeter in this baseline.

## Why login comes last

A greeter changes what appears on the primary virtual terminal during boot. A
broken Niri configuration, locker, PAM path, or greeter command can therefore
look like a machine that no longer boots. The system is easier to recover when
manual TTY login, manual Niri startup, locking, and suspend have already been
tested independently.

greetd authenticates and launches sessions. tuigreet is only its terminal user
interface. swaylock authenticates an already logged-in user. swayidle does not
authenticate anybody: it watches idle and sleep events and invokes the locker
or display-power commands.

## Prerequisites

- Chapters 01 through 10 are complete.
- Niri starts reliably with `niri-session -l` from TTY.
- Waybar, Fuzzel, Mako, swaybg and the polkit agent start once.
- Manual suspend and resume from chapter 06 work.
- A second TTY can be reached with `Ctrl+Alt+F3` and accepts the `neon` login.
- The local `niri-dotfiles` clone contains the reviewed chapter 11 files and is
  clean before they are copied.

Do not enable greetd until every explicit checkpoint says to continue.

## Audit existing login and lock components

```bash
systemctl status display-manager.service --no-pager
systemctl is-enabled greetd.service 2>&1
pgrep -a swaylock
pgrep -a swayidle
pacman -Q greetd greetd-tuigreet swaylock swayidle 2>&1
```

On the clean path, there is no active display manager and none of the four
packages is installed. If GDM, SDDM, LightDM, greetd, DankGreeter, swaylock, or
another idle daemon is already configured, stop. Do not layer this chapter on
top of an existing login or lock lifecycle.

## Install without enabling the greeter

Read Arch News and perform one complete update transaction:

```bash
sudo pacman -Syu swaylock swayidle greetd greetd-tuigreet
```

Installation is not authorization to enable greetd. Confirm it remains off:

```bash
systemctl is-enabled greetd.service
```

The expected result is `disabled`.

## Stage 1 — prove the locker

From the `niri-dotfiles` root, preview and deploy only swaylock:

```bash
stow --simulate --verbose --no-folding --target="$HOME" swaylock
stow --verbose --no-folding --target="$HOME" swaylock
```

Stop on a conflict; do not use `--adopt`. Inspect the resulting link:

```bash
readlink -f ~/.config/swaylock/config
```

Lock manually from inside Niri:

```bash
swaylock -f
```

Confirm all of the following before proceeding:

- the desktop is completely covered;
- ordinary keys or pointer activity do not reveal it;
- an incorrect password is rejected;
- the normal user password unlocks it;
- after unlocking, applications and Waybar are still running.

This is the most important checkpoint. A pretty locker that cannot reliably
authenticate is not usable.

## Stage 2 — deploy the lifecycle

Validate the changed Niri configuration before restowing it:

```bash
niri validate --config niri/.config/niri/config.kdl
stow --restow --verbose --no-folding --target="$HOME" niri
niri validate
```

The new Niri startup command launches one foreground-aware swayidle process
with these actions:

| Event | Action |
| --- | --- |
| 300 seconds idle | Run `swaylock -f`. |
| 600 seconds idle | Ask Niri to power off the monitors. |
| Activity after monitor timeout | Ask Niri to power the monitors on. |
| systemd-logind `before-sleep` event | Start swaylock and wait until its lock surface is ready. |
| logind session-lock event | Run `swaylock -f`. |

The `-w` option makes swayidle wait for each command where ordering matters.
No timeout invokes suspend. This avoids surprising sleep during downloads,
compilation, video, or initial testing; lid and manual suspend remain explicit.

Save work, exit Niri, and start it manually again:

```bash
niri-session -l
```

Confirm one idle daemon exists:

```bash
pgrep -a swayidle
```

Test the new binding with `Super+Alt+L`. Then test monitor power without waiting
ten minutes:

```bash
niri msg action power-off-monitors
```

Move the pointer or press a key only after running this recovery command from a
second TTY if needed:

```bash
niri msg action power-on-monitors
```

The Niri IPC command normally belongs to the graphical user session, so a TTY
without its environment may not reach that session. This is why the ordinary
activity-resume path must be tested before greetd is enabled.

### Test suspend protection

Lock once manually, unlock, save work, and request suspend:

```bash
systemctl suspend
```

After resume, the locker must be visible before any desktop content. Unlock and
inspect the current boot:

```bash
journalctl -b --no-pager | grep -Ei 'suspend|sleep|swayidle|swaylock'
```

Repeat the test by closing and reopening the lid. If either path exposes the
desktop briefly, fails to resume, or returns to an unlocked session, stop here
and keep manual TTY startup. Do not compensate by enabling greetd.

## Stage 3 — configure greetd and tuigreet

Back up the packaged example before replacing it:

```bash
sudo cp --archive /etc/greetd/config.toml /etc/greetd/config.toml.before-niri
sudoedit /etc/greetd/config.toml
```

Set the complete file to:

```toml
[terminal]
vt = 1

[default_session]
command = "tuigreet --time --remember --remember-session --greeting 'Arch Linux' --cmd niri-session"
user = "greeter"
```

This remembers the last successful username and session choice, never the
password. It provides Niri as the default while leaving tuigreet's session
selection available. It does not enable autologin.

Inspect the exact effective file and executable paths:

```bash
sudo cat /etc/greetd/config.toml
command -v greetd tuigreet niri-session
getent passwd greeter
ls -l /usr/share/wayland-sessions
```

Do not add `dbus-run-session`, a manual `DISPLAY`, or
`xwayland-satellite`. `niri-session` owns the supported session setup.

### Preserve Secret Service unlock through greetd

Chapter 07 attached GNOME Keyring to `/etc/pam.d/login` for the original
console-login path. greetd uses its own PAM service file, so those
program-specific hooks do not automatically apply to a tuigreet login.

Back up the packaged greetd policy before changing it:

```bash
sudo cp --archive /etc/pam.d/greetd /etc/pam.d/greetd.before-gnome-keyring
sudoedit /etc/pam.d/greetd
```

Preserve every existing line. Add the authentication hook at the end of the
existing `auth` section:

```pam
auth       optional     pam_gnome_keyring.so
```

Add the session hook at the end of the existing `session` section:

```pam
session    optional     pam_gnome_keyring.so auto_start
```

The password-change hook in `/etc/pam.d/passwd` remains the one installed in
chapter 07; do not duplicate it in the greetd policy. Verify only the intended
additions and the module path:

```bash
sudo grep -n 'pam_gnome_keyring' /etc/pam.d/greetd /etc/pam.d/passwd
test -e /usr/lib/security/pam_gnome_keyring.so
```

Keep the current Niri session and the recovery TTY available. A syntax error
in any PAM policy can prevent a new login even though an already authenticated
session continues to work.

## Enable with a live recovery TTY

Open `Ctrl+Alt+F3`, log in as `neon`, and leave that shell open. Return to Niri,
save all work, and exit the compositor. From the authenticated recovery TTY:

```bash
sudo systemctl enable greetd.service
sudo systemctl start greetd.service
```

Switch to `Ctrl+Alt+F1`. tuigreet should appear. Log in and confirm Niri starts.
Do not reboot yet.

Verify the session from Kitty:

```bash
loginctl session-status
systemctl status greetd.service --no-pager
pgrep -a niri
pgrep -a swayidle
systemctl --user is-active gnome-keyring-daemon.service
busctl --user list | grep 'org.freedesktop.secrets'
```

The keyring service must be active and the Secret Service bus name must be
present without asking separately for the login-keyring password. If the
keyring stays locked, return to the recovery TTY and inspect the greetd PAM
lines before rebooting.

Then repeat:

- manual lock and unlock;
- suspend and resume;
- lid close and resume;
- Niri exit back to tuigreet;
- a second successful login.

Only after all tests succeed should you reboot once:

```bash
systemctl reboot
```

The expected sequence is firmware, systemd-boot, LUKS password, boot, tuigreet,
authentication, and Niri. Plymouth and graphical LUKS presentation remain a
later boot-polish project.

## Recovery

If tuigreet or Niri fails, switch to `Ctrl+Alt+F3`, log in, and disable greetd:

```bash
sudo systemctl disable --now greetd.service
sudo cp --archive /etc/greetd/config.toml.before-niri /etc/greetd/config.toml
sudo cp --archive /etc/pam.d/greetd.before-gnome-keyring /etc/pam.d/greetd
systemctl reboot
```

The machine then returns to ordinary TTY login. If only idle handling is faulty,
comment out the swayidle `spawn-at-startup` line in the tracked Niri file,
validate it, and restow the `niri` package. If swaylock itself is faulty:

```bash
stow --delete --verbose --target="$HOME" swaylock
```

Never disable PAM password authentication or add passwordless sudo rules to
make a greeter power menu work.

## Completion checklist

- [ ] swaylock rejects a bad password and accepts the user password.
- [ ] `Super+Alt+L` locks immediately.
- [ ] Five idle minutes lock and ten idle minutes power off displays.
- [ ] Activity restores displays.
- [ ] Manual and lid-triggered suspend resume behind the locker.
- [ ] greetd was enabled only after the previous checks passed.
- [ ] tuigreet starts Niri without autologin.
- [ ] A tuigreet login unlocks GNOME Keyring through greetd's PAM policy.
- [ ] Exiting Niri returns to tuigreet.
- [ ] `Ctrl+Alt+F3` remains a working recovery route.
- [ ] A full reboot completes successfully.

## Sources

- [greetd package](https://archlinux.org/packages/extra/x86_64/greetd/)
- [greetd-tuigreet package](https://archlinux.org/packages/extra/x86_64/greetd-tuigreet/)
- [ArchWiki: GNOME Keyring](https://wiki.archlinux.org/title/GNOME/Keyring)
- [swayidle package](https://archlinux.org/packages/extra/x86_64/swayidle/)
- [swaylock package](https://archlinux.org/packages/extra/x86_64/swaylock/)
- [tuigreet upstream documentation](https://github.com/tuigreet/tuigreet)
