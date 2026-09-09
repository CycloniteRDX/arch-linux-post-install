# 21 — Refine Waybar without changing its role

## Goal

Begin the advanced personalization series with the most visible and least
risky session surface: Waybar. Replace the general-purpose text baseline with
a compact, icon-led, full-width bar while keeping the established component
owners and recovery paths.

This chapter preserves the existing Midnight Circuit palette, Noto text,
Niri-aware workspaces, and essential system status. It deliberately removes
the window title, tray, and `Exit` button from the bar, keeps session exit and
locking in Niri's established bindings, and adds the two icon-font packages
required by the selected glyphs. It adds no daemon, shell, widget framework,
AUR package, systemd unit, privilege rule, or machine-specific output setting.

The first personalization pass keeps the current component owners. Its
reviewed sequence, including later consolidations, is:

1. Waybar presentation and interactions;
2. Niri window, overview, input, and motion details;
3. Kitty, Mako, Fuzzel, swaylock, and resume restoration — consolidated into
   chapter 23;
4. swaybg and static wallpaper presentation — accepted without automation;
5. tuigreet — completed in chapter 24;
6. Bash, Nano, Micro, and Vim — completed in chapter 25;
7. minimal RogueOS Plymouth refinement — completed in chapter 26;
8. GTK and Qt cross-application consistency;
9. complete visual validation and a stable dotfiles release.

Only after that sequence is attractive and hardware-validated will the project
compare replacements such as SwayNotificationCenter or another wallpaper
renderer. Replacement experiments must not be mixed into this first pass.

## Status and prerequisites

This chapter passed hardware validation on the target ThinkPad T14 Gen 1 AMD
on 2026-09-07. In `arch-linux-post-install`, publish
`post-install-21-v1` at the final chapter-21-only commit `536d61a`. In
`niri-dotfiles`, publish the same tag name at `b922d85`, the last clean
Waybar-only state, rather than at a later Niri commit. These are independent
references in two repositories. Chapter 22 then records the cumulative
finished desktop.

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
| Bar glyph rendering | Fontconfig with `otf-font-awesome` and `ttf-nerd-fonts-symbols-mono` |
| Network state | NetworkManager, observed by Waybar |
| Audio and microphone state | PipeWire/WirePlumber, controlled through `wpctl` |
| Brightness | Kernel backlight interface through `brightnessctl` |
| Power-profile state | TLP through the standard D-Bus interface provided by `tlp-pd` |
| Battery state | Kernel power-supply interface observed by Waybar |
| Calendar application | GNOME Calendar, launched from the clock |
| Session exit and manual lock | Existing Niri bindings, outside Waybar |
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

Confirm the installed tools inherited from earlier chapters, running instance,
deployed links, and user-session health before pulling the candidate:

```bash
pacman -Q \
  waybar jq noto-fonts btop networkmanager bluez-utils blueman \
  pavucontrol brightnessctl gnome-calendar tlp-pd
command -v \
  waybar jq btop nmcli nmtui bluetoothctl blueman-manager \
  gnome-calendar pavucontrol wpctl brightnessctl tlpctl swaylock
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

## Install the selected icon fonts

The revised configuration uses two explicit fallback families:

| Package | Font family used by Waybar | Purpose |
| --- | --- | --- |
| `otf-font-awesome` | `Font Awesome 7 Free` | Workspace dots and most status icons |
| `ttf-nerd-fonts-symbols-mono` | `Symbols Nerd Font Mono` | Material Design glyphs used by the brightness scale |

Both packages are in Arch's official `Extra` repository; no AUR helper is
involved. Install them as one complete upgrade transaction:

```bash
sudo pacman -Syu otf-font-awesome ttf-nerd-fonts-symbols-mono
```

Pacman's font hooks update the Fontconfig cache. Verify the packages and the
family names consumed by `style.css`:

```bash
pacman -Q otf-font-awesome ttf-nerd-fonts-symbols-mono
command -v fc-match fc-list
fc-match -f '%{family}\n' 'Font Awesome 7 Free'
fc-match -f '%{family}\n' 'Symbols Nerd Font Mono'
fc-list : family | grep -F 'Font Awesome 7 Free'
fc-list : family | grep -F 'Symbols Nerd Font Mono'
```

Each `fc-match` result and `fc-list` search must name the requested family.
Stop before reloading Waybar if either search is empty; a fallback text font
can keep the process alive while still displaying replacement boxes.

## Review the Waybar change

Chapter 21 changes only these deployed files:

```text
waybar/.config/waybar/config.jsonc
waybar/.config/waybar/style.css
```

Documentation files may change in the same repositories. The only system
change is installation of the two official icon-font packages above; no Niri,
Mako, Fuzzel, swaylock, wallpaper, GTK, Qt, service, or boot configuration
changes.

The visual contract is:

- a compact full-width Midnight Circuit surface at the top of the output;
- 26 px configured height, 3 px vertical margins, and 4 px horizontal margins;
- a subtle 1 px cyan border and 7 px outer radius;
- Niri's dynamic workspaces at the left as small Font Awesome circles;
- a fixed-center US-style 12-hour clock;
- CPU, memory, temperature, network, Bluetooth, microphone, speaker,
  brightness, power profile, and battery at the right;
- cyan for focus and hover, fuchsia for performance mode, yellow for warnings,
  red for critical states, and green for power-saving or charging states;
- muted disconnected, disabled, and muted states;
- no window-title, tray, or session-action module;
- one Waybar process with the same module contract on every output.

The interaction contract adds only:

| Action | Result |
| --- | --- |
| Click clock | Open GNOME Calendar |
| Click CPU or memory | Open btop in Kitty |
| Left-click network | Toggle the NetworkManager Wi-Fi radio |
| Right-click network | Open `nmtui-connect` in Kitty |
| Left-click Bluetooth | Toggle the BlueZ controller power |
| Right-click Bluetooth | Open Blueman Manager |
| Right-click speaker or microphone | Open pavucontrol |
| Left-click speaker or microphone | Toggle the corresponding default endpoint mute |
| Scroll speaker | Adjust output volume by 5% in this historical checkpoint |
| Scroll microphone or brightness | Adjust the established control by 5% |
| Hover power profile | Show the profile exposed by `tlp-pd` |
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

1. The compact full-width surface is visually separated from the wallpaper by
   its margin and subtle border.
2. The outer border, radius, height, and module alignment remain consistent.
3. The center clock remains geometrically centered and is not pushed by the
   right-side status block.
4. Dynamic workspace dots appear and disappear with Niri's real workspace
   state; the focused dot remains distinct.
5. CPU, RAM, temperature, network, Bluetooth, volume, microphone, brightness,
   power profile, and battery remain readable without clipping.
6. Tooltips use the same surface, border, foreground, and spacing.
7. No missing-glyph square appears. All text must use Noto Sans and every icon
   must resolve through Font Awesome 7 Free or Symbols Nerd Font Mono.
8. There is no unexplained blank allocation for a removed title, tray, or
   `Exit` module.

Take a second private screenshot under similar conditions. Compare alignment,
contrast, density, and unused space rather than accepting the change merely
because it is new.

If an external display or dock is available, repeat the layout check there and
hotplug it once. This chapter deliberately does not add output names, scale,
resolution, or refresh-rate overrides.

## Validate every module and action

### Workspaces

Create several workspaces, change focus, move a window, and close the last
window on a temporary workspace. Confirm that Waybar follows Niri without stale
or duplicate dots.

Use Niri's own state for comparison:

```bash
niri msg workspaces
```

### Clock and calendar

Confirm that the clock uses the ordinary US-style weekday, numeric date, and
12-hour time with AM/PM. Left-click it; exactly one GNOME Calendar window
should open. The intentionally disabled tooltip and absent alternate format do
not need a second interaction.

### CPU, memory, and temperature

Compare the visible values with independent tools:

```bash
LC_ALL=C top -b -n1 | head -8
free -h
sensors
```

The refresh intervals are deliberately modest: CPU every 2 seconds, memory and
temperature every 5 seconds. The bar must not cause a visible idle CPU loop.
Click CPU and memory once each; both actions should open btop in Kitty. Close
each test window afterward.

### Network

Compare the module with NetworkManager, disconnect and reconnect once through
the established NetworkManager workflow, and inspect the tooltip:

```bash
nmcli general status
nmcli device status
nmcli -f NAME,TYPE,DEVICE connection show --active
```

The module must distinguish connected, disconnected, and disabled states.
Left-click once to disable Wi-Fi and once to enable it again; confirm that the
known connection returns. Right-click must open `nmtui-connect` in Kitty.
Cancel it without changing saved connections. Chapter 21 does not add a
network applet or put credentials in its configuration.

### Bluetooth

Compare the module with BlueZ:

```bash
bluetoothctl show
systemctl is-active bluetooth.service
```

Left-click once to turn the controller off and once to turn it on again. The
module colour and state must follow BlueZ. Right-click must open Blueman
Manager. Do not remove an existing pairing merely to test the button.

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

### Power profile and battery

Confirm that the separate power-profile module is reading TLP's compatibility
daemon, not a competing power manager:

```bash
systemctl is-active tlp-pd.service
tlpctl get
pacman -Q power-profiles-daemon 2>&1
```

`tlp-pd.service` must be active, `tlpctl get` must report one of the three
expected profiles, and `power-profiles-daemon` must remain absent. Hover the
Waybar module and confirm that its tooltip agrees. This chapter does not add a
profile-changing click action.

Observe the battery once on AC and once on battery:

```bash
upower -e
upower -i "$(upower -e | grep -m1 '/battery_')"
```

Charging, plugged, or full state should be green; normal discharge should use
the normal foreground; and the established warning/critical thresholds remain
25% and 10%. Do not drain the battery merely to force a colour test. The power
profile and battery must remain separate modules.

### Session exit and lock remain outside Waybar

At the historical `post-install-21-v1` checkpoint, use `Super+Alt+L`;
swaylock must cover the session and accept both one wrong password and the
correct password. After saving all work, use `Super+Shift+E`, confirm Niri's
exit request, and log in again through the unchanged tuigreet. Chapter 22 later
changes the lock binding to `Super+Shift+L`. The absence of `Exit` in
Waybar is deliberate and must not remove either Niri binding.

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

- several dynamic workspaces and focus changes;
- Wi-Fi reconnect;
- Bluetooth off/on and Blueman Manager;
- audio and microphone mute/unmute;
- brightness adjustment;
- power-profile tooltip comparison with `tlpctl get`;
- AC connection and disconnection;
- lock/unlock;
- logout through Niri's binding and a fresh tuigreet login;
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

This rollback changes no component other than Waybar. The two official font
packages may remain installed: unused fonts do not start a process or change a
policy, and keeping them avoids destructive package removal during recovery.
To return to development after diagnosis:

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
- [ ] Only the two Waybar deployment files and the documented font packages changed.
- [ ] `otf-font-awesome` and `ttf-nerd-fonts-symbols-mono` are installed from official repositories.
- [ ] Fontconfig resolves `Font Awesome 7 Free` and `Symbols Nerd Font Mono` by those exact family names.
- [ ] `jq empty` accepts both the repository and deployed JSON configuration.
- [ ] Stow preview and deployment complete without adopting another file.
- [ ] Reload leaves exactly one Waybar process.
- [ ] No JSON, CSS, GTK, module, or crash error appears in the relevant logs.
- [ ] The compact full-width bar aligns correctly on the internal panel.
- [ ] Dynamic workspaces, clock, tooltips, and all selected status values remain usable.
- [ ] Calendar, btop, Wi-Fi, nmtui, Bluetooth, Blueman, pavucontrol, mute, volume, and brightness actions work.
- [ ] Niri's separate exit and lock bindings still work.
- [ ] Connected, disconnected, disabled, muted, profile, charging, warning, and critical states remain distinguishable where safely observable.
- [ ] A fresh tuigreet login starts the new Waybar exactly once.
- [ ] Suspend/resume returns behind swaylock with a healthy bar.
- [ ] The dotfiles clone remains clean after ordinary use.
- [ ] `post-install-21-v1` is created only after hardware validation.
- [ ] SwayNC, Eww, a new wallpaper daemon, output overrides, 48 Hz automation, calendar synchronization, and boot/login theming remain outside this chapter.

## Sources

- [Waybar configuration manual](https://man.archlinux.org/man/waybar.5.en)
- [Waybar styling manual](https://man.archlinux.org/man/waybar-styles.5.en)
- [Waybar Niri workspaces module](https://man.archlinux.org/man/waybar-niri-workspaces.5.en)
- [Arch package: otf-font-awesome](https://archlinux.org/packages/extra/any/otf-font-awesome/)
- [Arch package: ttf-nerd-fonts-symbols-mono](https://archlinux.org/packages/extra/any/ttf-nerd-fonts-symbols-mono/)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Restic backup command](https://restic.readthedocs.io/en/latest/040_backup.html)
- [Restic repository checks](https://restic.readthedocs.io/en/latest/045_working_with_repos.html)

## Next step

Continue with chapter 22, which records the Niri work that was deliberately
moved forward and validated alongside the final Waybar input refinements. The
subsequent component, tuigreet, and terminal-workflow stages are now recorded
through chapter 25. GTK and Qt consistency is the next open review; optional
Plymouth restyling follows only if it offers a concrete improvement.
