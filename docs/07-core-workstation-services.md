# 07 — Configure core workstation services

## Goal

Complete the services that ordinary desktop applications expect: usable audio,
Bluetooth management, automatic removable-media mounting, and a login-unlocked
Secret Service. Keep printing conditional until a real printer is present.

This chapter makes the following changes:

- installs `pavucontrol` for PipeWire routing and device-profile selection;
- installs BlueZ, its current command-line tools, and Blueman;
- enables the system Bluetooth service;
- installs UDisks, udiskie, GVfs, and the MTP backend;
- deploys one XDG autostart file from `niri-dotfiles` for udiskie;
- installs GNOME Keyring and libsecret;
- integrates GNOME Keyring with console-login PAM;
- verifies speakers, microphone discovery, Bluetooth, removable media, and the
  Secret Service;
- documents a conservative, optional CUPS path without enabling network
  printer discovery or sharing.

It does not install a final file manager, notification daemon, tray or status
bar, password-manager application, Bluetooth file-transfer server, network
share client, or printer stack when no printer is required.

## Prerequisites

- Chapters 01 through 06 are complete.
- Manual `niri-session -l` startup, Kitty, and TTY recovery still work.
- PipeWire, WirePlumber, polkit, and the MATE polkit agent passed chapter 05.
- Firewalld remains enabled with no unexplained public services.
- The `niri-dotfiles` clone is clean and contains the reviewed chapter 07
  autostart package.
- A non-essential USB drive is available for the removable-media test.

Run ordinary commands as `neon` and use `sudo` only where shown. Do not test
with the Arch installation USB or with the only copy of important data.

## Resulting service model

| Area | Canonical result |
| --- | --- |
| Audio server | PipeWire from chapter 05 |
| Audio policy | WirePlumber from chapter 05 |
| Audio controls | `wpctl` and `pavucontrol` |
| Bluetooth stack | BlueZ with `bluetoothctl` |
| Bluetooth GUI | Blueman |
| Bluetooth audio | PipeWire; no PulseAudio server or Bluetooth add-on |
| Removable-media service | D-Bus-activated UDisks 2 |
| Automatic mounting | udiskie in the graphical user session |
| GTK file access | GVfs with MTP support for Android devices |
| Secret Service | GNOME Keyring with libsecret clients |
| Keyring unlock | Password-based PAM login; no automatic login |
| Printing | Conditional; disabled and uninstalled unless needed |

UDisks and GNOME Keyring are activation-based services. Do not enable their
system units manually. Bluetooth is the only new system service enabled by the
canonical procedure.

## Audit the existing service state

Confirm that the chapter 05 audio stack is still active from Kitty:

```bash
wpctl status
systemctl --user is-active \
    pipewire.service \
    pipewire-pulse.service \
    wireplumber.service
```

All three units must be active. Stop and repair chapter 05 if PipeWire has no
audio devices or any unit is failed.

Check for overlapping or legacy packages:

```bash
pacman -Q \
    pulseaudio \
    pulseaudio-bluetooth \
    pipewire-media-session \
    bluez-deprecated-tools 2>&1
```

All four packages should be absent. This project uses `bluetoothctl`, not the
deprecated `hciconfig` or `hcitool` utilities.

Inspect the Bluetooth radio without changing it:

```bash
rfkill list bluetooth
lsmod | grep -E '^btusb|^bluetooth'
systemctl is-enabled bluetooth.service 2>&1
systemctl is-active bluetooth.service 2>&1
```

The T14's controller should be listed. `disabled` and `inactive` are expected
before this chapter. A hard block must be resolved with the ThinkPad radio or
firmware control; software cannot override it.

## Install the service clients and integrations

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then perform one complete upgrade transaction:

```bash
sudo pacman -Syu \
    pavucontrol \
    bluez \
    bluez-utils \
    blueman \
    udisks2 \
    udiskie \
    gvfs \
    gvfs-mtp \
    gnome-keyring \
    libsecret
```

Read the transaction before accepting it. In particular, do not install
`pulseaudio-bluetooth` when pacman shows it as an optional Blueman dependency.
`pipewire-audio` already supplies Bluetooth-audio support for this system.

The packages have separate responsibilities:

| Package | Role |
| --- | --- |
| `pavucontrol` | Select audio profiles, defaults, ports, and per-app routing. |
| `bluez` | Provide the system Bluetooth daemon and protocol stack. |
| `bluez-utils` | Provide the supported `bluetoothctl` administration client. |
| `blueman` | Provide a desktop-independent Bluetooth manager and pairing agent. |
| `udisks2` | Expose storage operations over D-Bus and polkit. |
| `udiskie` | Automatically mount removable filesystems as the logged-in user. |
| `gvfs` | Make volumes and virtual filesystems available to GTK applications. |
| `gvfs-mtp` | Add Android and media-player access through MTP. |
| `gnome-keyring` | Implement `org.freedesktop.secrets` in the user session. |
| `libsecret` | Provide the client API and `secret-tool` test utility. |

Inspect unresolved package configuration files:

```bash
sudo pacdiff --output
```

Resolve every reported file before proceeding.

## Verify and test local audio

Confirm that the PipeWire-Pulse compatibility server owns the PulseAudio API:

```bash
pactl info | grep -E \
    'Server Name|Default Sink|Default Source'
wpctl status
```

`Server Name` should identify PulseAudio on PipeWire. Open the graphical mixer:

```bash
pavucontrol
```

In **Configuration**, select an analog duplex profile when the built-in
speakers and microphone are required. In **Output Devices** and **Input
Devices**, choose the intended ports and confirm that neither is unexpectedly
muted.

Test the built-in left and right speakers at a moderate volume:

```bash
wpctl set-volume @DEFAULT_AUDIO_SINK@ 40%
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
speaker-test -c 2 -t wav -l 1
```

The voice prompt should alternate between both channels. Stop immediately if
the level is uncomfortable.

Confirm that ALSA capture devices exist, then make a short disposable test:

```bash
arecord -l
arecord -d 5 -f cd /tmp/arch-post-install-microphone-test.wav
aplay /tmp/arch-post-install-microphone-test.wav
rm /tmp/arch-post-install-microphone-test.wav
```

The recording must contain the expected microphone input. The final command
removes only the known test file created immediately above.

Do not add WirePlumber rules, replace ALSA profiles, or delete its state merely
because a different output was selected once. Use `pavucontrol` or
`wpctl set-default ID` first; WirePlumber remembers deliberate default-device
choices.

Final Niri volume and microphone key bindings belong in `niri-dotfiles` after
the desktop-control components are selected.

## Enable and verify Bluetooth

Enable the BlueZ daemon:

```bash
sudo systemctl enable --now bluetooth.service
systemctl is-enabled bluetooth.service
systemctl is-active bluetooth.service
bluetoothctl show
```

The service must be enabled and active, and `bluetoothctl show` must display a
controller. Current BlueZ powers adapters on by default when its service starts
or the machine resumes.

If `rfkill list bluetooth` reports only a soft block, remove that specific
block and check again:

```bash
sudo rfkill unblock bluetooth
rfkill list bluetooth
bluetoothctl show
```

Do not use `rfkill unblock all`: it can silently change deliberate Wi-Fi or
WWAN state. Do not mask systemd's rfkill units; chapter 06 intentionally kept
normal radio-state persistence.

For normal daily use, power only the Bluetooth adapter down when it is not
needed:

```bash
bluetoothctl power off
bluetoothctl show | grep 'Powered:'
systemctl is-active bluetooth.service
nmcli radio wifi
```

The expected results are `Powered: no`, an active BlueZ service, and enabled
Wi-Fi. This changes the BlueZ adapter's `Powered` property; it neither stops
`bluetooth.service` nor disables the combined wireless controller. Turn the
adapter back on with:

```bash
bluetoothctl power on
bluetoothctl show | grep 'Powered:'
systemctl is-active bluetooth.service
nmcli radio wifi
```

The expected adapter state is now `Powered: yes`. Use these commands instead
of stopping BlueZ, unloading a kernel module, changing UEFI settings, or
running `rfkill ... all` merely to save battery.

Open the canonical graphical manager from Niri:

```bash
blueman-manager
```

Chapter 10 has not installed a notification daemon yet. At this checkpoint,
Blueman can therefore fall back to a small centered GTK notification window
after connecting or disconnecting a device. Under Niri that temporary window
may remain until it is closed with `Super+Q`. Read and close it; do not disable
Blueman's connection notifications merely to hide the fallback. Once the
notification daemon is installed, the same events should become ordinary
timed desktop notifications.

Pairing a device is not required to prove that the laptop controller works. If
one is available, use Blueman or the following supported console flow:

```text
bluetoothctl
power on
agent on
default-agent
scan on
devices
pair DEVICE_MAC_ADDRESS
trust DEVICE_MAC_ADDRESS
connect DEVICE_MAC_ADDRESS
scan off
quit
```

Replace `DEVICE_MAC_ADDRESS` with the address printed by `devices`. Confirm
every pairing code on both devices; do not trust an unidentified address.

For Bluetooth headphones, inspect the result with:

```bash
wpctl status
pavucontrol
```

PipeWire should add the headset without installing another audio server. Leave
Bluetooth PAN, DUN, discoverable-always, experimental BlueZ features, wake from
suspend, and OBEX file receiving disabled unless a real use case is documented.
Firewalld controls IP networking, not the Bluetooth protocol stack.

## Integrate GNOME Keyring with console login

GNOME Keyring is a Secret Service for applications; it is not a replacement
for a password manager. Its login collection should use the same password as
the `neon` account so PAM can unlock it without storing that password in a
script or dotfile.

Confirm that no prior integration is active:

```bash
grep -n 'pam_gnome_keyring' \
    /etc/pam.d/login \
    /etc/pam.d/passwd
```

No output is expected on the fresh project system. Back up both PAM files:

```bash
sudo cp -a /etc/pam.d/login /etc/pam.d/login.before-gnome-keyring
sudo cp -a /etc/pam.d/passwd /etc/pam.d/passwd.before-gnome-keyring
```

Edit the console-login policy:

```bash
sudo nano /etc/pam.d/login
```

Add this line at the end of the existing `auth` section:

```pam
auth       optional     pam_gnome_keyring.so
```

Add this line at the end of the existing `session` section:

```pam
session    optional     pam_gnome_keyring.so auto_start
```

Then edit the password-change policy:

```bash
sudo nano /etc/pam.d/passwd
```

Append:

```pam
password   optional     pam_gnome_keyring.so
```

This final hook keeps the login-keyring password synchronized when `passwd`
changes the account password.

Review only the added lines and confirm that the module exists:

```bash
grep -n 'pam_gnome_keyring' \
    /etc/pam.d/login \
    /etc/pam.d/passwd
test -e /usr/lib/security/pam_gnome_keyring.so
```

Do not close the current session yet. Press `Ctrl+Alt+F2`, log in as `neon` on
the second TTY, then return with `Ctrl+Alt+F1` or the function key containing
the first session. A successful second login proves that the PAM files still
parse. If it fails, restore both `.before-gnome-keyring` files from the original
session before rebooting.

Chapter 11 will add equivalent `auth` and `session` lines to
`/etc/pam.d/greetd` when greetd is actually installed. Do not create that file
early and do not configure automatic login or a blank keyring password.

## Deploy graphical automount from `niri-dotfiles`

UDisks starts on demand through D-Bus and uses polkit for privileged storage
operations. Do not enable `udisks2.service` manually and do not add `neon` to a
storage-related group.

Update the reviewed dotfiles clone:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git pull --ff-only
```

The tree must be clean before the pull. Review the new XDG autostart entry:

```bash
sed -n '1,120p' \
    autostart/.config/autostart/udiskie.desktop
```

It starts `udiskie --no-notify`: automatic mounting is enabled by default, but
notifications remain off until chapter 10 provides a notification daemon. It
does not request a tray icon before a StatusNotifier host exists.

Preview and deploy only the new Stow package:

```bash
stow \
    --simulate \
    --verbose \
    --no-folding \
    --target="$HOME" \
    autostart

stow \
    --verbose \
    --no-folding \
    --target="$HOME" \
    autostart
```

Verify the link:

```bash
ls -l ~/.config/autostart/udiskie.desktop
readlink -f ~/.config/autostart/udiskie.desktop
```

The resolved path must point into the `niri-dotfiles` clone. Niri's systemd
session starts the standard `xdg-desktop-autostart.target`, so no additional
Niri-specific startup command is required.

Make the user manager discover the newly deployed XDG autostart file before
testing another Niri session:

```bash
systemctl --user daemon-reload
```

The XDG autostart generator normally runs when the user manager starts. A full
reboot therefore also discovers the file, but merely closing and reopening
Niri can reuse the existing user manager. `daemon-reload` reruns its generators
and avoids making a reboot an accidental requirement.

## Restart the session and test removable media

Save work, exit Niri with `Super+Shift+E`, and start a fresh session:

```bash
niri-session -l
```

Open Kitty and confirm that exactly one udiskie daemon is running:

```bash
pgrep -a -x udiskie
udisksctl status
```

Insert a non-essential USB drive. Inspect its device name, filesystem, and
mountpoint:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS
findmnt -t vfat,exfat,ntfs,ntfs3,ext4
```

The removable filesystem should appear below `/run/media/neon/`. Do not test
automount by inserting another computer's encrypted system drive or by
unlocking an unknown LUKS container.

Before unplugging the test drive, unmount it through udiskie:

```bash
udiskie-umount -a
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
```

Wait until the mountpoint disappears and any device activity light stops. A
later file manager can invoke the same UDisks operations; it does not need to
run its own competing automounter.

Android MTP access is installed but receives its graphical test after chapter
09 selects the file manager. Network shares remain outside this chapter; no
SMB, NFS, Avahi, or discovery daemon is enabled.

## Optional: install local printing only when needed

Skip this section if neither ThinkPad currently needs a printer. A disabled,
unused print server is not part of the canonical workstation.

For a real USB printer or one configured through a known IPP URI, install the
local client and administration tools:

```bash
sudo pacman -Syu \
    cups \
    cups-filters \
    cups-pk-helper \
    system-config-printer
sudo systemctl enable --now cups.socket
```

Socket activation starts CUPS only when a local client needs it. Verify that
the local interface does not expose an unexplained network listener:

```bash
systemctl is-enabled cups.socket
systemctl is-active cups.socket
sudo ss -lntup | grep ':631'
```

Then run `system-config-printer` and add only the identified printer. Prefer
driverless IPP Everywhere when the device supports it. Do not enable printer
sharing, `cups-browsed.service`, Avahi, or an mDNS firewalld rule as part of
this local-only path. Network discovery and printer sharing require a separate
threat and firewall review in the handbook.

## Verify the completed service checkpoint

After the PAM change and XDG autostart deployment, reboot once and log in at
TTY with the normal account password:

```bash
sudo reboot
```

Start Niri manually, open Kitty, and run:

```bash
systemctl is-active bluetooth.service
systemctl --user is-active \
    pipewire.service \
    pipewire-pulse.service \
    wireplumber.service \
    gnome-keyring-daemon.service
pgrep -a -x udiskie
busctl --user list | grep -E \
    'org\.freedesktop\.secrets|org\.gnome\.keyring'
```

Bluetooth and all listed user services must be active; udiskie must have one
process; the user bus must expose the Secret Service. It is valid for one
keyring bus name to have a current owner while two related names appear as
`(activatable)`: those names are registered for D-Bus activation on demand,
not failed or disabled services. Do not enable extra keyring units manually;
the functional `secret-tool` test below is the decisive check.

Exercise the Secret Service with a disposable value:

```bash
printf '%s' 'temporary-test-secret' | secret-tool store \
    --label='Arch post-install verification' \
    project arch-post-install \
    purpose verification

secret-tool lookup \
    project arch-post-install \
    purpose verification

secret-tool clear \
    project arch-post-install \
    purpose verification
```

The lookup must print `temporary-test-secret`; `clear` must succeed. Never use
a real password, token, key, or recovery phrase for this test. Secret-store
contents and files below `~/.local/share/keyrings/` must never enter Git.

Finally confirm that no failed unit or new IP listener was hidden by the
desktop work:

```bash
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo ss -lntup
sudo firewall-cmd --get-active-zones
```

Investigate every failure or unexplained listener. Bluetooth, UDisks, and the
Secret Service use Bluetooth or D-Bus interfaces and do not require a new
firewalld service.

## Recovery

For ordinary battery saving, leave the daemon enabled and use
`bluetoothctl power off` as documented above. Only if the BlueZ daemon itself
causes a hardware or service problem should it be disabled while retaining the
installed diagnostic tools:

```bash
sudo systemctl disable --now bluetooth.service
```

If the new PAM integration prevents a login, use the still-open original TTY
or the verified Arch ISO and chroot procedure, then restore the backups:

```bash
sudo cp -a \
    /etc/pam.d/login.before-gnome-keyring \
    /etc/pam.d/login
sudo cp -a \
    /etc/pam.d/passwd.before-gnome-keyring \
    /etc/pam.d/passwd
```

If automatic mounting causes a problem, remove only the deployed autostart
link:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --delete --verbose --target="$HOME" autostart
```

This does not remove UDisks or the tracked source file. Manual mounting remains
available through `udisksctl`.

## Completion checklist

- [ ] PipeWire, PipeWire-Pulse, and WirePlumber remain active.
- [ ] `pavucontrol` sees the built-in output and input profiles.
- [ ] Both speakers and a short microphone recording work.
- [ ] BlueZ is enabled and `bluetoothctl show` sees the T14 controller.
- [ ] No deprecated BlueZ or overlapping PulseAudio package is installed.
- [ ] Blueman opens and can pair a deliberately selected device.
- [ ] UDisks remains D-Bus activated rather than manually enabled.
- [ ] The udiskie XDG autostart file is deployed from `niri-dotfiles`.
- [ ] Exactly one udiskie daemon runs in a fresh Niri session.
- [ ] A test USB filesystem mounts below `/run/media/neon/` and unmounts safely.
- [ ] GNOME Keyring unlocks after password-based console login.
- [ ] A disposable `secret-tool` value can be stored, read, and removed.
- [ ] No secret or generated keyring state is tracked by Git.
- [ ] Printing remains absent, or local CUPS socket activation was explicitly
      selected for an identified printer.
- [ ] No new unexplained IP listener, firewalld rule, or failed unit exists.
- [ ] Manual Niri startup and TTY recovery still work after reboot.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: PipeWire](https://wiki.archlinux.org/title/PipeWire)
- [ArchWiki: WirePlumber](https://wiki.archlinux.org/title/WirePlumber)
- [ArchWiki: Bluetooth](https://wiki.archlinux.org/title/Bluetooth)
- [ArchWiki: Blueman](https://wiki.archlinux.org/title/Blueman)
- [ArchWiki: Udisks](https://wiki.archlinux.org/title/Udisks)
- [ArchWiki: File manager functionality](https://wiki.archlinux.org/title/File_manager_functionality)
- [ArchWiki: GNOME Keyring](https://wiki.archlinux.org/title/GNOME/Keyring)
- [ArchWiki: CUPS](https://wiki.archlinux.org/title/CUPS)
- [WirePlumber: `wpctl`](https://pipewire.pages.freedesktop.org/wireplumber/man/wpctl.html)
- [udiskie upstream documentation](https://github.com/coldfix/udiskie/wiki/Usage)
- [Niri upstream: Integrating niri](https://github.com/niri-wm/niri/wiki/Integrating-niri)

## Next step

Continue with chapter 08 to create the XDG user directories and install the
fonts, shell and console tools, archive support, clipboard utilities, and
small common tools needed by the daily workstation.
