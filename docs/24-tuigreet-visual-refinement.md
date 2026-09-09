# 24 — Refine tuigreet without changing the login boundary

## Goal

Replace the deliberately plain chapter 11 presentation with the validated
RogueOS tuigreet profile while preserving every security and recovery
boundary:

- greetd remains the only display manager and continues to own VT1;
- tuigreet still runs as the unprivileged `greeter` system account;
- PAM continues to authenticate the target user and unlock GNOME Keyring;
- `niri-session` remains the only selected graphical session;
- automatic login remains disabled;
- TTY3 remains the independent recovery path.

This chapter changes the login screen's layout, labels, colours, remembered
state, and optional background animation. It does not modify Niri dotfiles,
swaylock, the console font, PAM policy, power commands, or Plymouth.

## Status

The final configuration passed real hardware validation on the first Lenovo
ThinkPad T14 Gen 1 AMD on 2026-09-08 with:

```text
greetd 0.10.3-2
greetd-tuigreet 0.11.1-2
```

The validated checks include an active and enabled greetd service, a successful
password login, a registered Wayland user session through the `greetd` PAM
service, `niri-session` startup, and successful GNOME Keyring unlock. This is a
system-only refinement. Its checkpoint is `post-install-24-v1` in
`arch-linux-post-install`; it does not require a matching `niri-dotfiles` tag.

## Prerequisites and recovery

Before changing the greeter:

- chapters 00 through 23 are complete;
- chapter 11 login, logout, PAM, keyring, and TTY3 recovery tests pass;
- `post-install-23-v1` is available as the previous documented checkpoint;
- only one display manager is enabled;
- the current graphical session has no unsaved work;
- an authenticated shell can be opened on TTY3 with `Ctrl+Alt+F3`.

Inspect the installed versions and current owner:

```bash
pacman -Q greetd greetd-tuigreet
systemctl is-enabled greetd.service
systemctl is-active greetd.service
systemctl cat greetd.service
```

The packaged unit must remain unmodified. Do not add a second display manager
or a local unit override merely to style tuigreet.

## Preserve staged rollback files

Chapter 11 already created the original pre-Niri copy:

```text
/etc/greetd/config.toml.before-niri
```

Before the first visual change, preserve the working chapter 11 state:

```bash
sudo cp --archive \
  /etc/greetd/config.toml \
  /etc/greetd/config.toml.pre-tuigreet-customization
```

If the static presentation is tested before enabling the animation, preserve
that additional boundary as well:

```bash
sudo cp --archive \
  /etc/greetd/config.toml \
  /etc/greetd/config.toml.pre-matrix
```

These copies contain configuration, not credentials. Keep them root-owned and
non-executable. The validated target has mode `0644` and owner `root:root` for
all four files.

## Install the validated configuration

Keep the authenticated recovery TTY open. Edit the system file:

```bash
sudo nano /etc/greetd/config.toml
```

Replace its complete contents with:

```toml
[terminal]
# Run the greeter on the first virtual terminal.
vt = 1

[default_session]
# Tuigreet configuration:
# - do not remember the username
# - remember the selected session
# - hide password length completely
# - use a compact cyan/magenta theme
# - use a slow, understated Matrix background
command = "tuigreet --time --time-format '%d/%m/%Y  %H:%M' --remember-session --greeting 'Authorized personnel only' --custom-title 'RogueOS' --width 52 --window-padding 2 --container-padding 2 --prompt-padding 1 --greet-align center --theme 'border=cyan;text=white;time=cyan;container=black;title=magenta;greet=cyan;prompt=magenta;input=white;action=cyan;button=magenta' --background matrix --matrix-length 4,12 --matrix-speed 0.10,0.45 --matrix-colors '#FFFFFF,#5CE1E6,#2A3A50' --cmd niri-session"

user = "greeter"
```

Inspect the exact result and permissions:

```bash
sudo sed -n '1,200p' /etc/greetd/config.toml
sudo find /etc/greetd -maxdepth 1 -type f \
  -printf '%M %u:%g %p\n' | sort
```

The TOML file contains no password. Keep the complete tuigreet invocation on
one TOML string so greetd passes the quoted arguments as intended.

### Remove the username remembered by the old baseline

Chapter 11 used `--remember`, so the old username may still exist in tuigreet's
cache even though the new command no longer reads, displays, or updates it.
Inspect the cache without printing its contents:

```bash
sudo find /var/cache/tuigreet -maxdepth 1 -type f \
  -printf '%M %u:%g %f\n' | sort
```

Remove only the two username records:

```bash
sudo rm -f -- \
  /var/cache/tuigreet/lastuser \
  /var/cache/tuigreet/lastuser-name
```

Do not remove `lastsession` or `lastsession-path`; those files implement the
selected `--remember-session` behavior. Confirm the intended result:

```bash
sudo test ! -e /var/cache/tuigreet/lastuser
sudo test ! -e /var/cache/tuigreet/lastuser-name
sudo stat -c '%U:%G %a %n' /var/cache/tuigreet
```

The directory must remain owned by `greeter:greeter` so session persistence
continues to work.

## Understand the selected behavior

| Setting | Effect | Boundary preserved |
| --- | --- | --- |
| `--time` and `--time-format` | Show `DD/MM/YYYY  HH:MM` | Does not change system time or locale |
| `--remember-session` | Store the last successfully selected session | Does not remember a username or password |
| no `--remember` | Require the username after every greeter start | Avoids exposing the last account name |
| no `--asterisks` | Show no per-keystroke password feedback | Hides password length as well as contents |
| `--greeting` and `--custom-title` | Display the selected warning and RogueOS title | Presentation only |
| width and padding options | Produce a compact centered prompt | Does not alter the Linux console geometry |
| `--theme` | Apply named ANSI colours to prompt components | Limited by the active virtual console palette |
| `--background matrix` | Draw an animated backdrop while tuigreet is visible | Stops consuming greeter resources after login |
| Matrix length `4,12` | Use shorter streams than the upstream default | Presentation only |
| Matrix speed `0.10,0.45` | Use a slow fall rate | Presentation only |
| Matrix colours | Select white, cyan, and dark blue bands | Accepts explicit RGB colour values |
| `--cmd niri-session` | Keep Niri as the default session | Manual session selection can still override it |

`--remember-session` can override `--cmd` after a different session is chosen
successfully. On the validated machine, `/usr/share/wayland-sessions` contains
only `niri.desktop`, whose `Exec` value is also `niri-session`, so both paths
currently converge on the same supported command.

The animation is active only while tuigreet is drawing the login interface.
After a successful login, the service process list contains greetd while the
authenticated Niri session replaces the greeter process.

## Apply without sacrificing recovery

Do not restart greetd underneath a graphical session containing unsaved work.
Keep TTY3 authenticated, save all work, and exit Niri normally. Return to TTY3
and restart greetd only after confirming that the old Niri session ended:

```bash
pgrep -a niri
sudo systemctl restart greetd.service
```

Switch to VT1 with `Ctrl+Alt+F1`. Confirm the complete presentation, type one
incorrect password, and then log in normally. A reboot is not required for the
first test.

## Validate the resulting session

From the fresh Niri session:

```bash
systemctl is-enabled greetd.service
systemctl is-active greetd.service
loginctl show-session "$XDG_SESSION_ID" \
  -p Name \
  -p User \
  -p Type \
  -p Class \
  -p State \
  -p Service \
  -p Desktop \
  -p Leader
find /usr/share/wayland-sessions -maxdepth 1 -type f -printf '%f\n' | sort
sed -n '1,160p' /usr/share/wayland-sessions/niri.desktop
sudo journalctl -b -u greetd.service --no-pager
```

The validated result is:

- greetd is `enabled` and `active`;
- the target session has `Type=wayland`, `Class=user`, `State=active`, and
  `Service=greetd`;
- `niri.desktop` exists and launches `niri-session`;
- the wrong password is rejected and the correct password starts Niri;
- the login prompt does not reveal password length;
- exiting Niri returns to the styled tuigreet interface;
- TTY3 remains usable.

An empty `Desktop=` property is not a failure on this target. The decisive
logind evidence is the active user class, Wayland type, greetd service, and the
actual `niri-session` process. Desktop naming exported inside the graphical
environment is a separate property.

Because `/etc/greetd/config.toml` and `/etc/pam.d/greetd` are package backup
files with deliberate local policy, this check reports metadata and checksum
differences for both:

```bash
pacman -Qkk greetd
```

Those named backup-file differences are expected here. The final `0 altered
files` summary refers to the package's ordinary non-backup payload; it does not
cancel or deny the separately reported local configuration differences.

### Interpret the GNOME Keyring messages

The validated journal contains this successful sequence:

1. the technical `greeter` session reports that it could not unlock a login
   keyring;
2. the target user's authentication phase cannot yet locate the daemon control
   file and temporarily stashes the entered password;
3. PAM opens the target-user session;
4. the session hook starts or reaches GNOME Keyring and reports that it
   unlocked the login keyring.

The first message is harmless because the system account only draws tuigreet.
For the target user, the final `unlocked login keyring` line establishes that
the earlier messages were intermediate states rather than a failure. Keep the
chapter 11 PAM policy unchanged:

```pam
auth       optional     pam_gnome_keyring.so
session    optional     pam_gnome_keyring.so auto_start
```

## Roll back only the failed layer

If only the animation is unsuitable, restore the static styled configuration:

```bash
sudo cp --archive \
  /etc/greetd/config.toml.pre-matrix \
  /etc/greetd/config.toml
```

If the full visual refinement fails, restore the working chapter 11 state:

```bash
sudo cp --archive \
  /etc/greetd/config.toml.pre-tuigreet-customization \
  /etc/greetd/config.toml
```

Perform the restart from TTY3 only after the graphical session has ended:

```bash
sudo systemctl restart greetd.service
systemctl --no-pager --full status greetd.service
```

Do not weaken PAM, add autologin, or replace `niri-session` to repair a visual
option. Restore the known-good command first and inspect the journal.

## Completion checklist

- [ ] The installed versions and unmodified greetd unit are recorded.
- [ ] The three staged rollback files remain root-owned and readable.
- [ ] The final TOML matches the validated complete configuration.
- [ ] Username memory is disabled and session memory remains enabled.
- [ ] Stale `lastuser` and `lastuser-name` cache records are absent.
- [ ] Password entry reveals neither contents nor length.
- [ ] Date, title, greeting, compact layout, theme, and Matrix presentation are correct.
- [ ] A wrong password fails and the correct password starts Niri.
- [ ] The session is registered as an active Wayland user session through greetd.
- [ ] GNOME Keyring finishes with a successful unlock message.
- [ ] Exiting Niri returns to tuigreet and TTY3 remains usable.
- [ ] No change exists in `niri-dotfiles` for this system-owned refinement.
- [ ] `post-install-24-v1` is created at the final post-install documentation commit.

This configuration passed the complete login, session, keyring, and recovery
validation on the first target on 2026-09-08.

## Sources

- [tuigreet manual](https://man.archlinux.org/man/tuigreet.1.en)
- [tuigreet upstream documentation](https://github.com/tuigreet/tuigreet)
- [Arch package: greetd-tuigreet](https://archlinux.org/packages/extra/x86_64/greetd-tuigreet/)
- [greetd upstream](https://sr.ht/~kennylevinsen/greetd/)
- [GNOME Keyring PAM integration](https://wiki.gnome.org/Projects/GnomeKeyring/Pam)

## Next step

Deploy and validate the separate Bash, Nano, Micro, and Vim packages in chapter
25. After that terminal workflow is recorded, complete the GTK and Qt
consistency review before deciding whether to restyle Plymouth and running the
complete desktop validation.
