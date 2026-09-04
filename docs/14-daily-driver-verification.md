# 14 — Verify the daily driver

## Goal

Validate the complete ThinkPad installation as one system after all individual
chapters have passed. This chapter adds no packages, services, keys, boot
parameters, or dotfiles. It verifies that the layers already configured work
together and remain recoverable after updates, reboots, suspend cycles, and
ordinary desktop use.

The canonical machine for this record is `RogueThinkpad`, a Lenovo ThinkPad T14
Gen 1 with an AMD Ryzen 5 PRO 4650U, 16 GiB RAM, NVMe storage, LUKS2 over LVM,
ext4 root/home, a 16 GiB disk-swap fallback, zram, systemd-boot, signed UKIs,
Secure Boot, Niri, greetd, and the reviewed Stow configuration.

## Pass criteria

The machine is **daily-driver ready** only when:

- every critical checkpoint below passes;
- every warning has a written explanation or a tracked follow-up;
- no unexplained kernel, storage, authentication, resume, or graphics failure
  appears during the observation period;
- TTY, fallback UKI, ISO/chroot, and backup recovery paths remain usable;
- all four project repositories are clean and synchronized;
- no secret or generated runtime state entered Git.

Hibernation, TPM2 unlocking, Plymouth, automatic idle suspend, remote calendar
synchronization, and host-specific output tuning are outside the current
profile. Their absence is not a failed test.

## Prepare the verification record

Create a private local record outside every Git repository:

```bash
mkdir -p ~/Documents/System-Records
micro ~/Documents/System-Records/RogueThinkpad-final-verification.md
```

Record:

- date and local time;
- current kernel and Niri versions;
- each section's result: `PASS`, `WARN`, `FAIL`, or `NOT TESTED`;
- exact symptoms and journal timestamps for warnings or failures;
- hardware used for Bluetooth, USB and external-display tests;
- Restic snapshot ID and recovery-bundle verification date, but no passwords,
  private keys, UUID inventory, Wi-Fi details, or full logs containing secrets.

Store this private record in the user-data backup, not in the public
documentation repository.

## Phase 1 — establish a clean baseline

Start after a normal cold boot through the default UKI and tuigreet. Do not
begin immediately after changing configuration.

Capture versions and identity:

```bash
hostnamectl
uname -a
cat /etc/os-release
niri --version
pacman -Q linux systemd niri waybar greetd greetd-tuigreet swaylock swayidle
```

Confirm time and boot health:

```bash
timedatectl
systemctl is-system-running
systemctl --failed
systemd-analyze time
systemd-analyze critical-chain
```

`systemctl is-system-running` should report `running`. A failed unit is not
automatically fatal, but it must be identified and either fixed or explicitly
explained before completion.

Review high-priority messages from the current boot:

```bash
journalctl -b -p warning..alert --no-pager
journalctl --user -b -p warning..alert --no-pager
dmesg --level=err,warn
```

Firmware, ACPI and drivers sometimes emit harmless warnings. Do not suppress
them to obtain empty output. Classify each recurring message by subsystem and
confirm whether it corresponds to an observable failure.

## Phase 2 — verify the boot and trust chain

```bash
bootctl --esp-path=/boot status
bootctl --esp-path=/boot list
cat /proc/cmdline
findmnt /boot
ls -lh /boot/EFI/Linux
sudo sbctl status
sudo sbctl verify
bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
```

Required results:

- UEFI and Secure Boot are enabled;
- `arch-linux.efi` is the default and current entry;
- both UKIs exist and are recognized;
- the command line opens the LUKS UUID as `cryptlvm` and uses
  `/dev/mapper/vg0-root`;
- `/boot` is the 1 GiB FAT32 ESP on `/dev/nvme0n1p1`;
- systemd-boot paths and both UKIs verify as signed;
- an unsigned `/boot/vmlinuz-linux` report remains expected because it is an
  input to the signed UKIs, not a directly booted entry.

### Boot the fallback UKI

Save work and reboot. Open the systemd-boot menu and select
`arch-linux-fallback.efi` once. After login:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo sbctl verify
```

Confirm the fallback entry booted, LUKS unlocked, root/home mounted, networking
worked, and Niri started. Reboot again into the default UKI before continuing.
Failure of the fallback image is a critical failure even when the normal image
works.

### Test a complete shutdown

After returning to the default UKI, save work and run:

```bash
systemctl poweroff
```

Wait until the power LED and fan are fully off, then perform a cold start.
Confirm the firmware, systemd-boot, LUKS prompt, default UKI, tuigreet and Niri
sequence completes normally. A reboot alone does not prove the complete
power-off path.

## Phase 3 — verify storage, memory, and filesystems

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo cryptsetup status cryptlvm
sudo pvs
sudo vgs
sudo lvs -a -o +devices
findmnt /
findmnt /home
findmnt /boot
swapon --show --output=NAME,TYPE,SIZE,USED,PRIO
zramctl
systemctl status systemd-zram-setup@zram0.service --no-pager
systemctl status fstrim.timer --no-pager
sudo fstrim --listed-in /etc/fstab:/proc/self/mountinfo --verbose --dry-run
```

Required results:

- `/dev/nvme0n1p2` backs the open LUKS2 mapping `cryptlvm`;
- `vg0` contains exactly root, home and swap;
- root and home are ext4 and `/boot` is vfat;
- zram has priority 100 and the 16 GiB disk swap has lower priority;
- zswap is disabled;
- `fstrim.timer` is enabled and the dry run identifies supported filesystems.

Confirm filesystem capacity and inode headroom:

```bash
df -hT / /home /boot
df -ih / /home /boot
```

No filesystem should be close to full. In particular, verify the ESP has enough
room for both UKIs and future updates.

## Phase 4 — perform one controlled full update

Read the current [Arch News](https://archlinux.org/news/) and inspect pending
updates without partially upgrading:

```bash
checkupdates
```

Then run the supported complete transaction:

```bash
sudo pacman -Syu
sudo pacdiff --output
```

Resolve every `.pacnew` deliberately. Inspect whether the transaction rebuilt
the kernel or UKIs:

```bash
journalctl -b --since "10 minutes ago" --no-pager
ls -lh --time-style=long-iso /boot/EFI/Linux
sudo sbctl verify
niri validate
```

If the kernel, systemd, mkinitcpio, microcode, sbctl, or bootloader changed,
reboot before performing desktop tests. After reboot, repeat phases 1 and 2.
An update that leaves unsigned or missing UKIs is a critical failure.

## Phase 5 — network and firewall

```bash
nmcli general status
nmcli device status
nmcli connection show --active
ip -brief address
ip route
resolvectl status
ping -c 4 1.1.1.1
ping -c 4 archlinux.org
```

The IP ping distinguishes basic connectivity from DNS resolution. Do not place
the full `nmcli connection show` output in a public issue; profile names and
network details may be sensitive.

Verify the firewall and listening services:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all --zone=public
sudo nft list ruleset
sudo ss -lntup
systemctl is-enabled sshd.service
systemctl is-active sshd.service
```

Required results:

- firewalld is running with `public` as default;
- the zone retains only the reviewed DHCPv6 client service and no SSH opening;
- NetworkManager remains the only network manager;
- `sshd` is disabled and inactive;
- every listening socket has an understood owner and purpose.

Test Wi-Fi reconnect after:

1. disconnecting and reconnecting from NetworkManager;
2. enabling and disabling airplane mode once;
3. suspending and resuming once;
4. moving between the normal access point and a phone hotspot if available.

The earlier concern about radio automation is why `tlp-rdw` remains absent.
Any repeated post-resume Wi-Fi lockup is a critical daily-driver issue.

## Phase 6 — power, battery, thermals, and suspend

```bash
sudo tlp-stat -s
sudo tlp-stat -b
sudo tlp-stat -p
tlpctl list
tlpctl get
sensors
brightnessctl info
```

Confirm TLP/`tlp-pd` is the only power-profile implementation and that the
ThinkPad charge thresholds are 75/80. Test all three profiles through
`tlpctl performance`, `tlpctl balanced`, and `tlpctl power-saver`, then restore
the normal source-based automatic selection with `sudo tlp start`.

Run three suspend cycles:

1. `systemctl suspend` while connected to AC;
2. lid close on battery;
3. suspend after several active applications, audio, Wi-Fi and Bluetooth are in
   use.

For each cycle confirm:

- swaylock appears before desktop content;
- display, keyboard, touchpad and TrackPoint recover;
- Wi-Fi reconnects;
- audio devices remain available;
- brightness controls work;
- battery state and TLP profile remain coherent;
- no second Waybar, Mako, swayidle or polkit agent appears.

Inspect the cycles:

```bash
journalctl -b --no-pager | grep -Ei 'suspend|resume|sleep|amdgpu|wifi|wlan|swayidle|swaylock'
```

Hibernation must not appear in `swapon`, logind, greetd or idle policy. Do not
test hibernation in this profile.

## Phase 7 — Niri session and input

```bash
niri validate
niri msg outputs
niri msg workspaces
niri msg windows
loginctl session-status
```

Record the exact built-in-panel modes, including the three decimal refresh
digits, but do not edit the portable config during this verification. Confirm:

- US keyboard layout and repeat behavior;
- tap-to-click and disable-while-typing;
- TrackPoint pointing and middle-button scrolling at defaults;
- three-finger horizontal and vertical gestures;
- focus, move, width, height, workspace, floating, tabbed, overview, fullscreen,
  screenshot, launcher, close, lock and exit bindings;
- Waybar workspaces and focused-window title follow Niri;
- Kitty loads the tracked palette;
- Fuzzel, Mako and swaybg each have exactly one process.

```bash
pgrep -a waybar
pgrep -a mako
pgrep -a swaybg
pgrep -a swayidle
pgrep -af polkit-mate-authentication-agent
```

## Phase 8 — audio, microphone, camera, and notifications

```bash
wpctl status
systemctl --user status pipewire.service pipewire-pulse.service wireplumber.service --no-pager
```

Test:

- speakers and headphones;
- volume up/down/mute and visible Waybar changes;
- built-in microphone recording and microphone mute key;
- the ThinkPad microphone-mute LED follows the actual mute state;
- a browser video with audio;
- webcam access in a local browser test or application, then close it;
- `notify-send "Final verification" "Mako is working"`;
- a short GNOME Calendar event produces one reminder.

Do not retain a test recording containing private conversation. A missing mic
LED is not equivalent to a muted microphone; verify actual PipeWire state.

## Phase 9 — Bluetooth and removable media

```bash
systemctl status bluetooth.service --no-pager
bluetoothctl show
```

Pair or reconnect one real device, use it briefly, disconnect it, and reconnect
after suspend. If it is an audio device, confirm WirePlumber selects sensible
profiles and the built-in audio returns after disconnection.

Insert a non-critical USB storage device and verify:

- udiskie mounts it once under the user session;
- Nautilus displays it;
- a file can be copied to and read back from it;
- unmount/eject completes before physical removal;
- no root ownership or unexpected automount process remains.

```bash
pgrep -a udiskie
lsblk -o NAME,PATH,FSTYPE,LABEL,SIZE,MOUNTPOINTS,TRAN
```

Use disposable test data. Do not experiment on the only backup disk.

## Phase 10 — applications, MIME associations, and Secret Service

Verify the default handlers:

```bash
xdg-mime query default x-scheme-handler/http
xdg-mime query default inode/directory
xdg-mime query default application/pdf
xdg-mime query default image/png
xdg-mime query default text/plain
xdg-mime query default text/calendar
```

Open representative files from Nautilus and from Kitty with `xdg-open`:

- web link in Firefox;
- directory in Nautilus;
- PDF in Papers;
- image in Loupe;
- text in GNOME Text Editor;
- audio/video in Celluloid;
- archive in File Roller;
- `.ics` file in GNOME Calendar;
- Writer, Calc and Impress documents in LibreOffice Still.

Confirm Micro, Vim, Git and GitHub CLI start, while remembering that their full
development configuration remains handbook work.

Test one application that stores a disposable secret through GNOME Keyring,
log out and back in, confirm retrieval, then delete the test secret through the
application. Never expose or commit the keyring files.

## Phase 11 — portals, screen sharing, and file chooser

```bash
systemctl --user status xdg-desktop-portal.service xdg-desktop-portal-gnome.service --no-pager
journalctl --user -b -u xdg-desktop-portal.service -u xdg-desktop-portal-gnome.service --no-pager
```

From Firefox or another reviewed application:

1. invoke the portal file chooser and cancel after confirming it opens;
2. share a single window in a trusted local WebRTC test;
3. share one monitor;
4. stop sharing and confirm the indicator disappears;
5. verify protected content is not accidentally shared if any future
   `block-out-from` rule is introduced.

Do not publish a screen stream or use a site that records it. Portal success is
an interaction test, not permission to disclose desktop contents.

## Phase 12 — external display

If a monitor or television is available, connect it and record:

```bash
niri msg outputs
```

Test hot-plug, cursor travel, window/workspace movement, mixed scaling if
applicable, audio-output selection, unplugging, and reconnection after suspend.
Niri should preserve workspace placement where possible and recover the
built-in panel without a compositor restart.

If no external display is available, mark this section `NOT TESTED`, not
`PASS`. Host-specific output blocks remain deferred until the actual modes and
desired physical arrangement are known.

## Phase 13 — maintenance and repository reproducibility

```bash
systemctl status paccache.timer fstrim.timer --no-pager
sudo pacdiff --output
pacman -Qdt
```

`pacman -Qdt` lists orphan candidates; it does not authorize removal. Review
dependencies before any later cleanup.

Check all project repositories:

```bash
for project_repo in arch-linux-runbook arch-linux-post-install arch-linux-handbook niri-dotfiles; do git -C "$HOME/Projects/CycloniteRDX/$project_repo" status --short --branch; done
```

Each must show `main...origin/main` without changes below it. Then repeat the
chapter 13 clean-target Stow test and confirm no broken links.

## Phase 14 — backup and recovery confirmation

Connect the verified encrypted backup disk and run:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad check --read-data-subset=5%
sudo sh -c 'cd /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery && sha256sum --check SHA256SUMS'
```

Replace the placeholder mount point with the verified real mount point. Confirm
the second critical copy and Restic-password recovery method still exist. The
ISO/chroot rehearsal from chapter 12 must already have passed; repeating it is
necessary after major storage or boot redesign, not on every verification run.

Create one final Restic snapshot after the system is declared stable:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad backup /home/neon --one-file-system --exclude '/home/neon/.cache' --exclude '/home/neon/.local/share/Trash' --tag daily-driver-ready
```

## Phase 15 — observation period

Use the laptop normally across at least three separate sessions, including one
session of several hours. Include:

- battery and AC operation;
- browser video and ordinary audio;
- several lock/unlock cycles;
- at least two additional suspend/resume cycles;
- Wi-Fi reconnect;
- file editing and Git work;
- USB or Bluetooth use when available.

At the end of the period:

```bash
systemctl --failed
journalctl -p warning..alert --since "24 hours ago" --no-pager
journalctl --user -p warning..alert --since "24 hours ago" --no-pager
sudo journalctl -k --since "24 hours ago" --no-pager | grep -Ei 'nvme|i/o error|filesystem|ext4|amdgpu'
sensors
```

Do not treat all warnings as equal. A repeated corrected firmware warning may
be informational; an NVMe media error, I/O error, filesystem error, GPU reset,
authentication failure, or incomplete suspend requires investigation before
sign-off.

## Final classification

Use exactly one outcome:

| Result | Meaning |
| --- | --- |
| `READY` | Every critical check passes; remaining notes are understood and non-blocking. |
| `READY WITH FOLLOW-UPS` | Core safety and daily use pass; explicitly non-critical improvements remain. |
| `NOT READY` | Any boot, storage, trust, backup, authentication, suspend, network, or recurring stability failure remains unresolved. |

Current follow-ups that do not block `READY` include the reviewed AUR workflow,
Qt theming, advanced modular-dotfiles polish, battery-only automatic idle
suspend, Plymouth, TPM2-bound unlock, calendar synchronization, a 48 Hz battery
profile, and host-specific output overrides. They must not be smuggled into
this verification as last-minute changes. The reviewed GTK, icon, cursor,
palette, and wallpaper baseline is applied only after this readiness gate, in
chapter 15.

## Final checklist

- [ ] Normal and fallback UKIs boot with Secure Boot enabled.
- [ ] A complete shutdown and cold start pass.
- [ ] LUKS, LVM, filesystems, zram, disk swap and TRIM match the profile.
- [ ] A complete update preserves signed bootability.
- [ ] Firewall, DNS, Wi-Fi reconnect and hotspot tests pass.
- [ ] Battery thresholds, profiles, thermals, lid and repeated suspend pass.
- [ ] Niri, input, all bindings, greetd, locking and idle behavior pass.
- [ ] Audio, mic LED, camera, notifications and calendar reminder pass.
- [ ] Bluetooth and removable-media tests pass or are explicitly not tested.
- [ ] MIME handlers, applications, Secret Service and portals pass.
- [ ] External display passes or is explicitly not tested.
- [ ] All repositories and the Stow reconstruction are clean.
- [ ] Restic, recovery-bundle checksums, second critical copy and ISO rehearsal pass.
- [ ] The observation period has no unexplained critical errors.
- [ ] The private record contains the final classification.

## Sources

- [ArchWiki: System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [systemd system state](https://man.archlinux.org/man/systemctl.1.en)
- [journalctl](https://man.archlinux.org/man/journalctl.1.en)
- [bootctl](https://man.archlinux.org/man/bootctl.1.en)
- [sbctl](https://man.archlinux.org/man/sbctl.8.en)
- [NetworkManager nmcli](https://man.archlinux.org/man/nmcli.1.en)
- [firewall-cmd](https://man.archlinux.org/man/firewall-cmd.1.en)
- [Restic documentation](https://restic.readthedocs.io/en/latest/)
