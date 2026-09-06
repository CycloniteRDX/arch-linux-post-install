# 21 — Refine Waybar without changing its role

## Goal

Begin the advanced personalization series with the most visible and least
risky session surface: Waybar. Keep every established module and action while
turning the full-width opaque strip into three restrained floating islands for
the left, center, and right module groups.

This chapter preserves the existing Midnight Circuit palette, ordinary Noto
fonts, Niri-aware workspaces, system-status modules, tray, and session action.
It adds no daemon, shell, widget framework, icon font, AUR package, systemd
unit, privilege rule, or machine-specific output setting.

The first personalization pass keeps the current component owners. The planned
order is:

1. Waybar presentation and interactions;
2. Fuzzel;
3. Mako;
4. swaylock;
5. swaybg and wallpaper presentation;
6. Niri window, overview, and motion details;
7. Kitty plus GTK and Qt consistency;
8. tuigreet;
9. Plymouth;
10. complete visual validation and a stable dotfiles release.

Only after that sequence is attractive and hardware-validated will the project
compare replacements such as SwayNotificationCenter or another wallpaper
renderer. Replacement experiments must not be mixed into this first pass.

## Status and prerequisites

This chapter is reviewed and awaits hardware validation on the target ThinkPad
T14 Gen 1 AMD. Do not create `post-install-21-v1` until every interaction,
state colour, reload, logout/login, and rollback test below passes.

Before applying it:

- chapters 00 through 20 are complete and chapter 20 is hardware-validated;
- `post-install-20-v1` exists in `arch-linux-post-install`;
- the four project repositories are clean and synchronized;
- the deployed dotfiles state matches the known-good `post-install-18-v2`
  checkpoint, whether the clean checkout is detached there or `main` still
  points to the same commit;
- Waybar starts exactly once from Niri and every current module works;
- TTY3 and manual `niri-session` recovery remain available;
- the chapter 12 encrypted backup disk and Restic password are available.

Run the Arch commands as `neon` from Kitty unless a step says otherwise. Save
work before the logout test. The repository changes themselves are prepared
and published separately from Windows; this procedure never asks the installed
machine to commit from a detached tag.

## Preserve the ownership model

| Concern | Owner after this chapter |
| --- | --- |
| Compositor, workspaces, windows and outputs | Niri |
| Bar layout and status presentation | Waybar |
| Network state | NetworkManager, observed by Waybar |
| Audio and microphone state | PipeWire/WirePlumber, controlled through `wpctl` |
| Brightness | Kernel backlight interface through `brightnessctl` |
| Battery state | Kernel power-supply interface observed by Waybar |
| Tray protocol | Waybar tray module and the applications that publish items |
| Calendar application | GNOME Calendar, launched from the clock |
| Session exit | Niri's own quit action |
| Locking and authentication | swaylock plus PAM |

Waybar displays state and dispatches narrow user actions. It does not become a
network manager, audio server, power-policy daemon, notification daemon,
locker, greeter, or desktop shell.

## Record the pre-personalization checkpoint

This is the one full user-data checkpoint taken before the visual series. Later
component chapters rely primarily on Git and their individual rollback paths.

### Verify the four project states

Set the established project directory and inspect every clone:

```bash
project_root=/home/neon/Projects/CycloniteRDX
for repo in \
  arch-linux-runbook \
  arch-linux-post-install \
  arch-linux-handbook \
  niri-dotfiles
do
  test -d "$project_root/$repo/.git"
  git -C "$project_root/$repo" status --short --branch
  git -C "$project_root/$repo" log -1 --oneline --decorate
done
```

Every branch must be clean. Confirm the last hardware-validated system and
dotfiles references explicitly:

```bash
git -C "$project_root/arch-linux-post-install" \
  show --no-patch --oneline --decorate post-install-20-v1
git -C "$project_root/niri-dotfiles" \
  show --no-patch --oneline --decorate post-install-18-v2
```

Stop if either reference is missing or an unexpected commit follows the known
work without an explanation.

### Open the encrypted external backup disk

Connect the dedicated chapter 12 disk and identify its persistent whole-disk
path:

```bash
ls -l /dev/disk/by-id/usb-*
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,SERIAL,TRAN
```

Assign the exact whole-disk entry, never a remembered `/dev/sdX` name. Replace
the deliberately invalid value:

```bash
backup_disk=/dev/disk/by-id/usb-REPLACE_WITH_THE_EXACT_WHOLE_DISK_ID
backup_partition="${backup_disk}-part1"
test -b "$backup_disk"
test -b "$backup_partition"
readlink -f "$backup_disk"
lsblk -d -o NAME,PATH,SIZE,TYPE,MODEL,SERIAL,TRAN,RM \
  "$(readlink -f "$backup_disk")"
```

The resolved target must be the expected USB disk and never the internal NVMe.
Unlock the LUKS partition once, using either Nautilus or:

```bash
udisksctl unlock -b "$backup_partition"
```

Discover the unlocked mapping rather than guessing its name:

```bash
backup_mapping="$(
  lsblk -nrpo PATH,TYPE "$backup_partition" |
  awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
printf 'Unlocked mapping: %s\n' "$backup_mapping"
```

Nautilus may already have mounted it. Query first and mount only when needed:

```bash
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
if [ -z "$backup_mount" ]; then
  udisksctl mount -b "$backup_mapping"
  backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
fi
test -n "$backup_mount"
findmnt --target "$backup_mount"
df -hT "$backup_mount"
```

The source must be the external decrypted mapping, the filesystem must be
ext4, and the mount must be read-write.

### Create and verify the Restic checkpoint

Resolve the existing repository and make sure it is on the mounted external
filesystem:

```bash
restic_repo="$backup_mount/restic-rogue-thinkpad"
test -d "$restic_repo"
findmnt --target "$restic_repo"
```

Create a new snapshot. Restic asks for its password interactively:

```bash
restic --repo "$restic_repo" backup /home/neon \
  --one-file-system \
  --exclude '/home/neon/.cache' \
  --exclude '/home/neon/.local/share/Trash' \
  --tag pre-personalization \
  --tag post-install-20-v1
```

The command must finish without unreadable files or snapshot errors. Inspect
and sample-check the result:

```bash
restic --repo "$restic_repo" snapshots \
  --tag pre-personalization \
  --tag post-install-20-v1
restic --repo "$restic_repo" stats latest
restic --repo "$restic_repo" check --read-data-subset=5%
```

Record the snapshot ID privately. Do not add the Restic password, device UUIDs,
or a complete machine inventory to Git.

### Close the disk completely

Rediscover the mapping and mount in the same shell, flush writes, and close the
stack in reverse order:

```bash
backup_mapping="$(
  lsblk -nrpo PATH,TYPE "$backup_partition" |
  awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
test -n "$backup_mount"
sync
udisksctl unmount -b "$backup_mapping"
udisksctl lock -b "$backup_partition"
udisksctl power-off -b "$(readlink -f "$backup_disk")"
```

Every closing command must succeed before disconnecting the cable. If the disk
is busy, inspect it rather than forcing the unmount:

```bash
sudo lsof +f -- "$backup_mount"
```

## Audit the working Waybar baseline

Confirm the installed tools, running instance, deployed links, and user-session
health before pulling the candidate:

```bash
pacman -Q waybar jq noto-fonts
command -v waybar jq gnome-calendar pavucontrol wpctl brightnessctl swaylock
waybar --version
waybar_count=$(pgrep -xc waybar || true)
printf 'Waybar processes: %s\n' "$waybar_count"
test "$waybar_count" -eq 1
readlink -f ~/.config/waybar/config.jsonc
readlink -f ~/.config/waybar/style.css
systemctl --user --failed --no-pager
```

Both links must resolve inside the expected `niri-dotfiles/waybar` package.
Stop if Waybar has a second startup owner or the live files are untracked
copies.

Capture the old appearance privately with one screenshot. It is useful for a
real before/after comparison, but it does not belong in the public repository
unless it is deliberately reviewed and stripped of personal information.

## Select the chapter 21 candidate

The first hardware-validation run uses the reviewed commit on `main`. The
immutable `post-install-21-v1` tag is created only after this chapter passes:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git switch main
git pull --ff-only
git status --short --branch
git log -1 --oneline --decorate
```

Stop on local changes, a non-fast-forward update, or a detached branch. During
future clean installations, replace this candidate step with:

```bash
git switch --detach post-install-21-v1
git describe --tags --exact-match
```

That tag must not be used until the project records hardware validation.

## Review the Waybar change

Chapter 21 changes only these deployed files:

```text
waybar/.config/waybar/config.jsonc
waybar/.config/waybar/style.css
```

Documentation files may change in the same repositories, but no Niri, Mako,
Fuzzel, swaylock, wallpaper, GTK, Qt, system, or boot configuration changes.

The visual contract is:

- transparent layer surface with three raised module islands;
- 10 px horizontal and 8 px upper breathing room;
- 38 px bar height with compact controls;
- rounded 12 px group shells and 8 px interactive states;
- cyan for time and active workspaces;
- fuchsia only for the session action;
- yellow for warnings, red for critical states, and green for charging;
- muted window title and muted audio states;
- ordinary text labels without Nerd Font coupling;
- the same modules and one Waybar process on every output.

The interaction contract adds only:

| Action | Result |
| --- | --- |
| Click clock | Open GNOME Calendar |
| Right-click speaker or microphone | Open pavucontrol |
| Left-click speaker or microphone | Toggle mute, unchanged |
| Scroll speaker, microphone, or brightness | Adjust the established control, unchanged |
| Left-click `Exit` | Request Niri session exit, unchanged |
| Right-click `Exit` | Lock with swaylock |
| `SIGUSR1` | Toggle bar visibility |
| `SIGUSR2` | Reload Waybar |

Inspect the complete source and the exact difference from the last validated
dotfiles checkpoint:

```bash
sed -n '1,260p' waybar/.config/waybar/config.jsonc
sed -n '1,320p' waybar/.config/waybar/style.css
git diff --check post-install-18-v2..HEAD
git diff --stat post-install-18-v2..HEAD
git diff post-install-18-v2..HEAD -- \
  waybar/.config/waybar/config.jsonc \
  waybar/.config/waybar/style.css
```

Validate the JSON before deployment:

```bash
jq empty waybar/.config/waybar/config.jsonc
```

No output and exit status zero are required. Waybar validates its GTK CSS only
when it loads the stylesheet, so runtime log inspection remains part of this
chapter.

## Preview and deploy only Waybar

Preview the Stow reconciliation:

```bash
stow --restow --simulate --verbose --no-folding \
  --target="$HOME" waybar
```

It may relink the two established Waybar targets. It must not create links for
another component or adopt an untracked target. Deploy only after the preview
is understood:

```bash
stow --restow --verbose --no-folding --target="$HOME" waybar
readlink -f ~/.config/waybar/config.jsonc
readlink -f ~/.config/waybar/style.css
jq empty ~/.config/waybar/config.jsonc
```

Both links must resolve inside the current clone. Reload the one running
instance with Waybar's default `SIGUSR2` action:

```bash
pkill -SIGUSR2 -x waybar
sleep 2
waybar_count=$(pgrep -xc waybar || true)
printf 'Waybar processes after reload: %s\n' "$waybar_count"
test "$waybar_count" -eq 1
```

If the process disappears, do not repeatedly start it. Move to TTY3 if needed,
restore the previous tag using the rollback section, and inspect the session
log.

Check the recent journal for parser, CSS, GTK, module, or crash errors:

```bash
journalctl --user --since '5 minutes ago' --no-pager |
  grep -Ei 'waybar|json|css|gtk|error|warning|critical' || true
systemctl --user --failed --no-pager
```

Warnings need interpretation, but an invalid property, CSS parse error, missing
module, assertion, or terminated Waybar process fails the chapter.

## Validate the visible layout

Perform these checks on the internal 1920×1080 panel:

1. The left, center, and right islands are visually separated from the
   wallpaper and from each other.
2. The islands have consistent borders, radius, vertical alignment, and shadow.
3. The center clock remains geometrically centered and is not pushed by the
   right-side status block.
4. A long window title truncates instead of covering the clock or status
   modules.
5. Active, inactive, hovered, and urgent workspace states remain distinct.
6. CPU, RAM, temperature, network, volume, microphone, brightness, battery,
   tray, and `Exit` remain readable without clipped text.
7. Tooltips use the same surface, border, foreground, and spacing.
8. No missing-glyph square appears; the design must still work with Noto Sans
   and Noto Color Emoji only.

Take a second private screenshot under similar conditions. Compare alignment,
contrast, density, and unused space rather than accepting the change merely
because it is new.

If an external display or dock is available, repeat the layout check there and
hotplug it once. This chapter deliberately does not add output names, scale,
resolution, or refresh-rate overrides.

## Validate every module and action

### Workspaces and title

Create several workspaces, change focus, move a window, and trigger an urgent
application state if one is naturally available. Confirm that Waybar follows
Niri without stale dots or a title from another output.

Use Niri's own state for comparison:

```bash
niri msg workspaces
niri msg focused-window
```

### Clock and calendar

Left-click the clock. Exactly one GNOME Calendar window should open. Right-click
the clock to toggle its compact alternate date, then inspect the calendar
tooltip.

### CPU, memory, and temperature

Compare the visible values with independent tools:

```bash
LC_ALL=C top -b -n1 | head -8
free -h
sensors
```

The refresh intervals are deliberately modest: CPU every 2 seconds, memory and
temperature every 5 seconds. The bar must not cause a visible idle CPU loop.

### Network

Compare the module with NetworkManager, disconnect and reconnect once through
the established NetworkManager workflow, and inspect the tooltip:

```bash
nmcli general status
nmcli device status
nmcli -f NAME,TYPE,DEVICE connection show --active
```

The module must distinguish connected and offline states. It remains a status
surface; chapter 21 does not add a network applet or put credentials in its
configuration.

### Audio and microphone

Test left click, right click, and scrolling on both audio modules. Left click
must toggle the correct default endpoint; right click must open pavucontrol;
scrolling must change by 5% without exceeding the existing 100% cap.

Compare with:

```bash
wpctl get-volume @DEFAULT_AUDIO_SINK@
wpctl get-volume @DEFAULT_AUDIO_SOURCE@
```

Muted states must use the muted colour and remain readable. Close pavucontrol
after the test.

### Brightness

Scroll the brightness module in both directions and compare it with:

```bash
brightnessctl info
```

Do not test minimum brightness in a way that leaves the internal panel unusable.

### Battery and tray

Observe the battery once on AC and once on battery:

```bash
upower -e
upower -i "$(upower -e | grep -m1 '/battery_')"
```

Charging or full state should be green, normal discharge should use the normal
foreground, and the established warning/critical thresholds remain 25% and
10%. Do not drain the battery merely to force a colour test.

Open every available tray menu once. Empty tray space is acceptable when no
application publishes an item; placeholder icons or a crashed bar are not.

### Session action and lock

Right-click `Exit`. swaylock must cover the session and accept both one wrong
password and the correct password. After saving all work, left-click `Exit`,
confirm Niri's exit request, and log in again through the unchanged tuigreet.

After login:

```bash
test "$(pgrep -xc waybar || true)" -eq 1
test "$(pgrep -xc mako || true)" -eq 1
test "$(pgrep -xc swaybg || true)" -eq 1
niri validate
systemctl --user --failed --no-pager
```

Waybar must start once with the new presentation. Mako, swaybg, swaylock,
swayidle, greetd, tuigreet, and Plymouth remain unchanged.

## Observe the candidate

Use the bar for at least one ordinary work session including:

- multiple workspaces and long window titles;
- Wi-Fi reconnect;
- audio and microphone mute/unmute;
- brightness adjustment;
- AC connection and disconnection;
- at least one tray menu;
- lock/unlock;
- logout through the bar and a fresh tuigreet login;
- one suspend/resume cycle behind swaylock.

At the end, check:

```bash
pgrep -a waybar
ps -C waybar -o pid,etimes,%cpu,%mem,rss,cmd
journalctl --user --since '1 hour ago' --no-pager |
  grep -Ei 'waybar|json|css|gtk|error|warning|critical' || true
systemctl --failed --no-pager
systemctl --user --failed --no-pager
git -C ~/Projects/CycloniteRDX/niri-dotfiles status --short --branch
```

Runtime or application state must not have modified the tracked configuration.
Exactly one healthy Waybar process and a clean dotfiles clone are required.

## Roll back to the validated bar

If the bar crashes, overlaps, clips essential modules, consumes unreasonable
resources, or any interaction targets the wrong action, return only the
dotfiles checkout to the last validated configuration:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git switch --detach post-install-18-v2
git describe --tags --exact-match
stow --restow --verbose --no-folding --target="$HOME" waybar
jq empty ~/.config/waybar/config.jsonc
pkill -SIGUSR2 -x waybar
sleep 2
test "$(pgrep -xc waybar || true)" -eq 1
```

This rollback changes neither system packages nor any component other than
Waybar. To return to development after diagnosis:

```bash
git switch main
git pull --ff-only
```

Do not delete Mako, swaybg, swaylock, tuigreet, Plymouth, or any other package
to repair a Waybar presentation problem.

## Completion checklist

- [ ] The four project clones were clean before personalization.
- [ ] A tagged `pre-personalization` Restic snapshot was created and checked.
- [ ] The external backup disk was unmounted, locked, and powered off cleanly.
- [ ] Only the two Waybar deployment files changed.
- [ ] `jq empty` accepts both the repository and deployed JSON configuration.
- [ ] Stow preview and deployment complete without adopting another file.
- [ ] Reload leaves exactly one Waybar process.
- [ ] No JSON, CSS, GTK, module, or crash error appears in the relevant logs.
- [ ] The three visual islands align correctly on the internal panel.
- [ ] Workspaces, long titles, clock, tooltips, status values, and tray remain usable.
- [ ] Calendar, pavucontrol, mute, volume, brightness, exit, and lock actions work.
- [ ] Connected, disconnected, muted, charging, warning, critical, and urgent states remain distinguishable where safely observable.
- [ ] A fresh tuigreet login starts the new Waybar exactly once.
- [ ] Suspend/resume returns behind swaylock with a healthy bar.
- [ ] The dotfiles clone remains clean after ordinary use.
- [ ] `post-install-21-v1` is created only after hardware validation.
- [ ] SwayNC, Eww, a new wallpaper daemon, output overrides, 48 Hz automation, calendar synchronization, and boot/login theming remain outside this chapter.

## Sources

- [Waybar configuration manual](https://man.archlinux.org/man/waybar.5.en)
- [Waybar styling manual](https://man.archlinux.org/man/waybar-styles.5.en)
- [Waybar Niri workspaces module](https://man.archlinux.org/man/waybar-niri-workspaces.5.en)
- [Waybar Niri window module](https://man.archlinux.org/man/waybar-niri-window.5.en)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Restic backup command](https://restic.readthedocs.io/en/latest/040_backup.html)
- [Restic repository checks](https://restic.readthedocs.io/en/latest/045_working_with_repos.html)

## Next step

After hardware validation and publication of `post-install-21-v1`, personalize
Fuzzel using the same Midnight Circuit geometry and state hierarchy. Keep
Waybar selected while that next component is evaluated so visual differences
can be attributed to one change at a time.
