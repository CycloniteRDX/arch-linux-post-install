# 06 — Configure ThinkPad hardware and power

## Goal

Configure the hardware and power baseline for the ThinkPad T14 Gen 1 AMD,
then verify firmware support, battery care, suspend, lid behavior, thermals,
brightness controls, function keys, touchpad, and TrackPoint.

This chapter makes the following changes:

- installs `fwupd`, its UEFI capsule updater, and the metadata-refresh timer;
- prevents fwupd's optional peer-to-peer cache from exposing a local server;
- signs fwupd's package-owned EFI updater with the enrolled owner key;
- configures fwupd to launch that signed updater directly, without Shim;
- selects TLP as the only high-level power manager;
- installs `tlp-pd` for the standard three-profile D-Bus interface;
- enables TLP and its profile daemon;
- limits normal battery charging to 75–80 percent;
- makes suspend explicit for the lid and suspend key on battery and AC power;
- keeps a closed lid from suspending the laptop when docked or using multiple
  displays;
- restricts systemd's generic sleep operation to suspend;
- installs small tools for battery, thermal, brightness, and input inspection;
- verifies the ThinkPad hardware without replacing firmware-managed fan
  control.

It does not configure hibernation, automatic firmware installation, custom fan
curves, CPU undervolting, overclocking, `powertop --auto-tune`, automatic radio
switching, or final Niri media-key and input preferences.

## Prerequisites

- Chapters 01 through 05 are complete.
- The manual Niri session and the TTY recovery path both work.
- The system is fully updated and connected through NetworkManager.
- Firewalld is enabled with no unexplained public services.
- Secure Boot, systemd-boot, and both UKIs still pass the chapter 01 checks.
- The charger is available before any firmware update is applied.

Run the commands as `neon` and use `sudo` only where shown. Kitty may be used
for ordinary checks, but perform the first suspend test while physically next
to the laptop and with no unsaved work.

## Resulting policy

| Area | Canonical result |
| --- | --- |
| Firmware client | `fwupd` with LVFS metadata refreshes |
| Firmware installation | Explicit and operator-confirmed, never automatic |
| Firmware Secure Boot path | Owner-key-signed fwupd EFI updater; no Shim |
| Firmware peer-to-peer cache | Disabled; `passim.service` masked |
| Power manager | TLP only |
| Profile interface | `tlp-pd`, replacing `power-profiles-daemon` |
| Automatic profile | `performance` on AC, `balanced` on battery |
| Manual low-power profile | `power-saver` through `tlpctl` |
| Battery thresholds | Start at 75%, stop at 80% on `BAT0` |
| Suspend mode | Suspend to RAM; deep sleep expected on this model |
| Lid on battery or AC | Suspend |
| Lid while docked or using multiple displays | Ignore |
| Hibernation | Not configured |
| Thermal and fan control | Kernel, ThinkPad firmware, and platform profile |
| Radio automation | Not installed; systemd rfkill state remains intact |
| Input configuration | Verify hardware now; final preferences stay in `niri-dotfiles` |

TLP is chosen because it combines power profiles, event-driven laptop tuning,
and native ThinkPad charge-threshold support. `tlp-pd` exposes the same power
profile D-Bus interface expected by desktop components without running a
second policy manager.

## Audit the starting hardware state

Confirm the exact model, firmware, processor, and kernel:

```bash
hostnamectl
lscpu | grep -E 'Model name|CPU\(s\)'
uname -r
```

The model must be `ThinkPad T14 Gen 1`. The canonical machines contain either
the Ryzen 5 PRO 4650U or Ryzen 7 PRO 4750U. Stop and revisit the project profile
if the computer is different.

Inspect the available sleep states and the currently selected suspend variant:

```bash
cat /sys/power/state
cat /sys/power/mem_sleep
```

`mem` must be available. On the T14 Gen 1 AMD the second command should contain
`[deep]`, for example:

```text
s2idle [deep]
```

If it instead contains `[s2idle] deep`, enter UEFI setup and select
`Config` → `Power` → `Sleep State` → `Linux`, then boot Arch and check again.
Do not add a kernel command-line override while the firmware setting can select
the supported deep-sleep path.

Inspect the ThinkPad platform profiles and relevant kernel modules:

```bash
cat /sys/firmware/acpi/platform_profile_choices
cat /sys/firmware/acpi/platform_profile
lsmod | grep -E 'thinkpad_acpi|k10temp'
```

The available profiles should include `low-power`, `balanced`, and
`performance`. `thinkpad_acpi` supplies ThinkPad integration and `k10temp`
supplies AMD temperature readings. Do not install `tp_smapi`: this modern
ThinkPad uses the in-tree `thinkpad_acpi` driver and Secure Boot remains
enabled.

## Audit competing power managers

Check installed packages that can write overlapping CPU, platform, PCIe, USB,
or power-profile settings:

```bash
pacman -Q \
    tlp \
    tlp-pd \
    power-profiles-daemon \
    tuned \
    tuned-ppd \
    laptop-mode-tools \
    auto-cpufreq 2>&1
```

Also inspect their system services:

```bash
systemctl is-enabled \
    tlp.service \
    tlp-pd.service \
    power-profiles-daemon.service \
    tuned.service 2>&1
```

On the fresh project system these packages and services should be absent. Stop
if another power manager is already installed or enabled; do not allow two
tools to repeatedly overwrite the same kernel settings.

## Install the hardware and power tools

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then install the chapter packages in one complete upgrade transaction:

```bash
sudo pacman -Syu \
    fwupd \
    fwupd-efi \
    tlp \
    tlp-pd \
    upower \
    lm_sensors \
    brightnessctl \
    libinput-tools \
    wev
```

Read the complete transaction before accepting it. `fwupd-efi` is listed
explicitly because the Secure Boot procedure below uses its package-owned
updater. The transaction can also pull in support libraries such as `udisks2`,
`passim`, and BlueZ. That does not enable Bluetooth or select removable-media
behavior; those services are integrated deliberately in chapter 07.

The installed tools have distinct roles:

| Package | Role |
| --- | --- |
| `fwupd` | Discover and install firmware delivered through LVFS |
| `fwupd-efi` | Provide the EFI capsule updater that runs before Arch boots |
| `tlp` | Apply one coherent laptop power policy and battery care |
| `tlp-pd` | Expose TLP's performance, balanced, and power-saver profiles |
| `upower` | Report batteries and power devices to users and desktop tools |
| `lm_sensors` | Display kernel hardware-monitoring sensors |
| `brightnessctl` | Inspect and change display or keyboard-backlight brightness |
| `libinput-tools` | Identify and diagnose input devices below the compositor |
| `wev` | Display Wayland keyboard events inside Niri |

Inspect package-manager messages and unresolved configuration files:

```bash
sudo pacdiff --output
```

Resolve every reported `.pacnew`, `.pacsave`, or `.pacorig` file before
continuing.

## Keep fwupd private and activation-based

`fwupd.service` is D-Bus activated and must not be enabled manually. Its
optional Passim integration can republish downloaded metadata on TCP port
27500. This workstation has no reason to serve that cache to the local
network.

Set fwupd's peer-to-peer policy to nothing, then mask the cache service:

```bash
sudo fwupdmgr modify-config P2pPolicy nothing
sudo systemctl mask --now passim.service
```

Verify the local policy and service state:

```bash
sudo grep -n 'P2pPolicy' /etc/fwupd/fwupd.conf
systemctl is-enabled passim.service
sudo ss -lntup | grep 27500
```

The configuration must show `P2pPolicy=nothing`, the service must be `masked`,
and the final command must produce no output. Firewalld remains useful, but an
unneeded server should not listen merely because the firewall can block it.

Enable only the timer that refreshes firmware metadata:

```bash
sudo systemctl enable --now fwupd-refresh.timer
systemctl is-enabled fwupd-refresh.timer
systemctl list-timers --all fwupd-refresh.timer --no-pager
```

The timer downloads metadata; it does not install firmware automatically.

## Prepare fwupd for owner-key Secure Boot

The runbook enrolled local owner keys and signs systemd-boot and the UKIs with
`sbctl`. It does not install or use Shim. Firmware capsule updates therefore
need the same direct trust path: the package-owned fwupd EFI updater is signed
with the enrolled owner key, and fwupd is told to launch it directly.

First prove which package owns the unsigned updater:

```bash
pacman -Qo /usr/lib/fwupd/efi/fwupdx64.efi
```

The owner must be `fwupd-efi`. Stop if the file is missing or belongs to
anything else; do not sign an unexplained EFI executable.

Create a signed companion and save the input/output relationship in sbctl's
file database:

```bash
sudo sbctl sign --save --output /usr/lib/fwupd/efi/fwupdx64.efi.signed /usr/lib/fwupd/efi/fwupdx64.efi
```

Verify the result and its saved registration:

```bash
sudo sbctl verify /usr/lib/fwupd/efi/fwupdx64.efi.signed
sudo sbctl list-files
```

The signed file must verify successfully, and the saved file list must contain
the fwupd updater. The `--save` registration allows sbctl's pacman hook to
re-sign the output after a future `fwupd-efi` package upgrade replaces the
unsigned input. Chapter 02's post-upgrade `sudo sbctl verify` remains mandatory.

Now edit fwupd's existing configuration:

```bash
sudoedit /etc/fwupd/fwupd.conf
```

Preserve the existing `[fwupd]` section and its `P2pPolicy=nothing` setting.
Add this separate section:

```ini
[uefi_capsule]
DisableShimForSecureBoot=true
```

If `[uefi_capsule]` already exists, add only the setting inside it instead of
creating a duplicate section. This option does not disable Secure Boot or
relax signature verification. It tells fwupd that the trusted updater can be
launched directly rather than chainloaded through a nonexistent
`shimx64.efi`.

Inspect the relevant settings and restart the D-Bus-activated daemon so that
it reads the new configuration:

```bash
sudo grep -nE '^\[(fwupd|uefi_capsule)\]|^(P2pPolicy|DisableShimForSecureBoot)=' /etc/fwupd/fwupd.conf
sudo systemctl restart fwupd.service
```

The effective local configuration must contain both policies under their
respective sections:

```ini
[fwupd]
P2pPolicy=nothing

[uefi_capsule]
DisableShimForSecureBoot=true
```

Do not install Shim, copy another distribution's `shimx64.efi`, create a fake
symlink, disable Secure Boot, or set `OnlyTrusted=false`. Those actions would
replace or weaken the deliberately constructed owner-key trust path instead
of completing it.

## Inventory firmware before changing it

Refresh metadata and inspect every recognized device:

```bash
fwupdmgr refresh
fwupdmgr get-devices
fwupdmgr get-updates
```

The ThinkPad system firmware and embedded controller should normally be
recognized. `No updates available` is a successful result, even though
`fwupdmgr get-updates` can return exit status 2 when there was nothing to do.

Do not change fwupd's trusted-firmware policy, enable the testing remote, or
force an unsupported release merely to make an update appear. If the ThinkPad
is not recognized, record `fwupdmgr get-devices` and move the diagnosis to the
handbook instead of using a Lenovo firmware file intended for another model.

## Apply an offered firmware update safely

Skip this section when `fwupdmgr get-updates` reports no updates.

Before accepting an offered update:

1. Read its vendor, target device, current version, new version, and release
   notes.
2. Connect the Lenovo charger directly to the laptop.
3. Confirm that the battery is comfortably charged:

```bash
upower -d
```

4. Close applications and save all work.
5. Confirm that Secure Boot and the signed boot path are still healthy:

```bash
sudo sbctl status
sudo sbctl verify
sudo bootctl status
sudo sbctl verify /usr/lib/fwupd/efi/fwupdx64.efi.signed
```

Apply only the releases offered by the enabled stable remotes:

```bash
sudo fwupdmgr update
```

Read every prompt. A capsule update can be staged by creating a UEFI boot entry
named `Linux Firmware Updater` and selecting it once through `BootNext`. Before
accepting the reboot, inspect the staged state:

```bash
fwupdmgr check-reboot-needed
sudo sbctl status
sudo sbctl verify
sudo efibootmgr -v
```

Do not delete the firmware-updater entry or clear `BootNext`: both are part of
the pending update. Do not disconnect power, force a shutdown, close the lid,
or press the reset button while firmware is being written. Allow every
firmware-controlled reboot to finish even if the ThinkPad restarts more than
once.

Two specific pre-staging errors identify an incomplete Secure Boot setup:

| Error | Meaning and action |
| --- | --- |
| `fwupdx64.efi.signed cannot be found` | Repeat the package-ownership, sbctl signing, and verification steps above. |
| `Shim is required but was not found` | Confirm the `[uefi_capsule]` section, then restart `fwupd.service` before retrying. |

For any different EFI, Secure Boot, capsule, or firmware-signature error, stop
and preserve the exact output. Do not use `--force`, install Shim as a shortcut,
or sign a different executable merely because its name appears in an error.

After a successful firmware reboot, verify the result and the boot chain:

```bash
fwupdmgr get-history
hostnamectl
sudo sbctl status
sudo sbctl verify
sudo bootctl status
sudo efibootmgr -v
```

`fwupdmgr get-history` must report `Success` for the release. `BootNext` should
no longer exist because UEFI consumes it after one use. A persistent `Linux
Firmware Updater` entry may remain outside `BootOrder`; that is harmless and
allows fwupd to reuse it later. The normal `Linux Boot Manager` must remain in
`BootOrder`. Do not delete entries merely because their numbers differ between
machines.

Do not assume that a firmware update preserves every UEFI preference. Recheck
Secure Boot, Linux sleep mode, boot order, and any deliberately changed
firmware option.

## Enable TLP as the only power manager

Enable TLP and its profile daemon:

```bash
sudo systemctl enable --now tlp.service
sudo systemctl enable --now tlp-pd.service
sudo tlp start
```

Use TLP's own status command rather than treating the boot-time `tlp.service`
as a continuously running daemon:

```bash
sudo tlp-stat -s
tlpctl list
tlpctl get
```

`tlp-stat -s` must report TLP enabled and `tlp-pd` enabled and running. The
active profile should normally be `performance` on AC power and `balanced` on
battery power. TLP's shipped defaults remain in force; this chapter does not
invent CPU frequency limits, EPP values, PCIe rules, or USB exceptions before
the machines have shown a real problem.

Confirm that no competing profile daemon became active:

```bash
pacman -Q power-profiles-daemon tuned-ppd 2>&1
systemctl is-active power-profiles-daemon.service tuned.service 2>&1
```

Both packages should be absent and both services inactive or not found.

`tlp-rdw` is intentionally not installed. Do not mask
`systemd-rfkill.service` or `systemd-rfkill.socket`: automatic radio switching
is outside this profile, and preserving ordinary rfkill state avoids adding a
second source of Wi-Fi soft blocks.

## Configure ThinkPad battery thresholds

First let TLP report the detected battery-care driver and supported values:

```bash
sudo tlp-stat -b
```

The output should identify the `thinkpad` plugin, `thinkpad_acpi`, and support
for charge thresholds. Stop if `Supported features` does not include charge
thresholds; do not install a legacy out-of-tree module on this T14.

Check that the main configuration does not already set active thresholds:

```bash
grep -E '^[[:space:]]*(START|STOP)_CHARGE_THRESH' /etc/tlp.conf
```

No output is expected on the fresh system. Create one focused local drop-in:

```bash
sudoedit /etc/tlp.d/10-thinkpad-battery.conf
```

Add exactly:

```ini
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
```

Set ordinary root-owned permissions, apply the configuration, and read the
values back from the embedded controller:

```bash
sudo chmod 0644 /etc/tlp.d/10-thinkpad-battery.conf
sudo tlp start
sudo tlp-stat -c
sudo tlp-stat -b
cat /sys/class/power_supply/BAT0/charge_control_start_threshold
cat /sys/class/power_supply/BAT0/charge_control_end_threshold
```

The final values must be 75 and 80. The start threshold means that connecting
AC at 78% does not immediately recharge the battery; charging begins only
after it falls below 75%. The stop threshold limits normal charging to 80%.
The firmware's embedded controller performs the actual charging behavior, so
the thresholds can remain effective while Arch is shut down.

For a trip that needs maximum runtime, temporarily request a full charge:

```bash
sudo tlp fullcharge BAT0
```

The configured 75/80 thresholds return at the next boot or after
`sudo tlp setcharge`. Do not run `tlp recalibrate` as routine maintenance; it
is a diagnostic operation involving a deliberate full discharge and recharge.

## Verify power and thermal integration

Inspect TLP's processor, platform, battery, and device summary:

```bash
sudo tlp-stat -p
sudo tlp-stat -b
upower -d
sensors
```

Then compare TLP's active profile with the firmware platform profile:

```bash
tlpctl get
cat /sys/firmware/acpi/platform_profile
```

Unplug the charger, wait a few seconds, and run both commands again. TLP should
automatically select the battery profile. Reconnect AC and verify that it
returns to the AC profile.

The three profiles can later be selected manually without `sudo`:

```bash
tlpctl performance
tlpctl balanced
tlpctl power-saver
```

After experimenting, return to automatic source-based selection:

```bash
sudo tlp start
```

Do not enable a `powertop` auto-tune service on top of TLP. Do not install
`thinkfan`, `thermald`, `ryzenadj`, or an AUR tuning daemon for the canonical
system. The firmware, `thinkpad_acpi`, AMD kernel drivers, and the selected
platform profile already coordinate the normal thermal and fan policy.

The initial temperature check is simply:

```bash
sensors
journalctl -b -k --no-pager | grep -iE 'thermal|overheat|throttl'
```

No output from the journal filter is normal. Investigate repeated thermal or
throttling warnings before changing control software. Do not run
`sensors-detect` unless an expected sensor is missing and the probe has first
been reviewed for this hardware.

## Make suspend and lid behavior explicit

Create a local logind drop-in:

```bash
sudo install -d -m 0755 /etc/systemd/logind.conf.d
sudoedit /etc/systemd/logind.conf.d/70-thinkpad-suspend.conf
```

Add:

```ini
[Login]
SleepOperation=suspend
HandleSuspendKey=suspend
HandleHibernateKey=ignore
HandleLidSwitch=suspend
HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore
```

This keeps the normal lid behavior identical on battery and external power,
while a docked or multi-display workstation may continue using its external
screen with the lid closed. `SleepOperation=suspend` also ensures that a
generic systemd sleep request cannot select suspend-then-hibernate.

Set the file permissions and inspect the merged configuration:

```bash
sudo chmod 0644 /etc/systemd/logind.conf.d/70-thinkpad-suspend.conf
systemd-analyze cat-config systemd/logind.conf
```

The final effective values must match the drop-in. Do not restart
`systemd-logind` from the graphical session because doing so can disrupt active
sessions. Save work and reboot instead:

```bash
sudo reboot
```

After booting, verify deep suspend and the merged policy again:

```bash
cat /sys/power/mem_sleep
systemd-analyze cat-config systemd/logind.conf
```

## Test suspend before relying on it

Chapter 11 has not installed the screen locker yet. Until then, suspend and
resume do not provide an unattended physical-access boundary. Perform these
tests while supervising the laptop and do not leave it suspended in public.

Save all work and request manual suspend:

```bash
systemctl suspend
```

Wake the laptop with the power button. After resume, check the suspend unit,
power policy, network, and failed units:

```bash
journalctl -b -u systemd-suspend.service --no-pager
sudo tlp-stat -s
nmcli general status
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

The system must resume to the same session, TLP must remain enabled, and
NetworkManager must reconnect. Investigate a failed unit instead of hiding it.

Now test the lid switch twice:

1. Disconnect AC, close the lid, wait for the sleep indicator, open the lid,
   and wake the laptop.
2. Connect AC and repeat the same test.

Both cases must suspend. A dock or second display is a separate test: with
that hardware connected, closing the lid should be ignored by logind.

If suspend hangs and a forced shutdown becomes unavoidable, boot Arch and
inspect the previous boot before adding workarounds:

```bash
journalctl -b -1 -u systemd-suspend.service --no-pager
journalctl -b -1 -k --no-pager
```

Do not enable hibernation, add `resume=`, switch sleep variants, or disable a
driver based on a single unexplained failure.

## Verify brightness and ThinkPad function keys

Inspect the available brightness devices:

```bash
brightnessctl --list
brightnessctl info
```

The internal display should normally use the AMD backlight device. Test a
small reversible change:

```bash
brightnessctl set 5%-
brightnessctl set +5%
```

Inside the manual Niri session, display Wayland key events for one minute:

```bash
timeout 60s wev
```

Keep the `wev` window focused and press the ThinkPad function row. Useful
events include:

| Keys | Expected role |
| --- | --- |
| `F1`–`F3` | Speaker mute and volume |
| `F4` | Microphone mute |
| `F5`–`F6` | Display brightness |
| `F7` | Display selection |
| `Fn+4` | Suspend key |
| `Fn+Space` | Keyboard backlight, handled by firmware |

`wev` should report the corresponding `XF86` key symbols for keys that reach
the compositor. The timeout closes the diagnostic automatically even though
the minimal Niri bootstrap does not yet contain a general close-window
binding.

This chapter proves that the key events reach the compositor; it does not
duplicate user bindings in the system repository. Brightness, volume, and
microphone bindings belong in `niri-dotfiles` and are deployed and tested in
chapter 10.

## Verify touchpad and TrackPoint

List the input devices and their libinput capabilities:

```bash
sudo libinput list-devices
```

Identify one internal touchpad and one TrackPoint. Do not copy a device name
or path from another ThinkPad; names can differ with firmware and component
supplier.

Observe live events, stopping the command with `Ctrl+C`:

```bash
sudo libinput debug-events
```

Test the following in the Niri session:

- pointer motion and physical click on the touchpad;
- two-finger touchpad scrolling;
- pointer motion with the TrackPoint;
- left, middle, and right TrackPoint buttons;
- scrolling by holding the middle TrackPoint button while moving the stick;
- automatic touchpad suppression while typing.

No system-wide libinput override is added when the defaults work. Preferences
such as tap-to-click, natural scrolling, acceleration, or per-machine keyboard
layout belong in the Niri configuration and are introduced only after their
desired behavior is chosen.

## Final service and exposure audit

Confirm the selected services and timers:

```bash
systemctl is-enabled \
    tlp.service \
    tlp-pd.service \
    fwupd-refresh.timer

systemctl is-active tlp-pd.service fwupd-refresh.timer
sudo tlp-stat -s
```

Confirm that optional services have not been enabled accidentally:

```bash
systemctl is-enabled \
    power-profiles-daemon.service \
    tuned.service \
    bluetooth.service \
    passim.service 2>&1
```

`power-profiles-daemon`, tuned, and Bluetooth must be disabled, inactive, or
not found at this stage. Passim must be masked.

Recheck sockets and the firewall after fwupd's new dependencies have been
installed:

```bash
sudo ss -lntup
sudo firewall-cmd --state
sudo firewall-cmd --zone=public --list-all
```

There must be no Passim listener on port 27500 and no unexplained new public
firewall service. Chapter 07 will enable Bluetooth deliberately and then repeat
the service audit.

## Completion checklist

- [ ] The machine is a supported ThinkPad T14 Gen 1 AMD.
- [ ] Deep suspend is selected in firmware and reported as `[deep]`.
- [ ] The ThinkPad platform profiles and kernel integration are present.
- [ ] `fwupd` recognizes the expected firmware devices.
- [ ] `fwupd-refresh.timer` is enabled, but firmware installation is manual.
- [ ] Peer-to-peer firmware caching is disabled and Passim is masked.
- [ ] The package-owned fwupd EFI updater is signed and saved in sbctl's database.
- [ ] fwupd is configured for direct owner-key launch without Shim.
- [ ] Secure Boot and both UKIs still verify after any firmware update.
- [ ] Firmware history, `BootNext`, and the final UEFI boot entries are checked.
- [ ] TLP is the only installed and enabled high-level power manager.
- [ ] `tlp-pd` is active and exposes three profiles.
- [ ] TLP changes automatically between its AC and battery profiles.
- [ ] `BAT0` reports active 75/80 charge thresholds.
- [ ] systemd's generic sleep operation is restricted to suspend.
- [ ] Manual suspend resumes with networking and TLP working.
- [ ] Lid close suspends on both battery and AC power.
- [ ] Docked lid behavior is explicitly ignored.
- [ ] Hibernation remains unconfigured.
- [ ] Sensors show plausible readings without a custom fan daemon.
- [ ] Brightness can be adjusted with `brightnessctl`.
- [ ] The function row generates the expected hardware or Wayland events.
- [ ] Touchpad and TrackPoint motion, buttons, and scrolling work.
- [ ] No Passim listener or unexplained public firewall exception exists.
- [ ] No system or user unit is unexpectedly failed.

## Sources

- [ArchWiki: Lenovo ThinkPad T14 (AMD) Gen 1](https://wiki.archlinux.org/title/Lenovo_ThinkPad_T14_(AMD)_Gen_1)
- [ArchWiki: fwupd](https://wiki.archlinux.org/title/Fwupd)
- [ArchWiki: Laptop](https://wiki.archlinux.org/title/Laptop)
- [ArchWiki: Power management](https://wiki.archlinux.org/title/Power_management)
- [ArchWiki: Suspend and hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate)
- [ArchWiki: TLP](https://wiki.archlinux.org/title/TLP)
- [ArchWiki: Backlight](https://wiki.archlinux.org/title/Backlight)
- [ArchWiki: TrackPoint](https://wiki.archlinux.org/title/TrackPoint)
- [TLP upstream: Arch Linux installation](https://linrunner.de/tlp/installation/arch.html)
- [TLP upstream: Usage](https://linrunner.de/tlp/usage/index.html)
- [TLP upstream: Battery care](https://linrunner.de/tlp/settings/battery.html)
- [fwupd upstream: Basic usage and Passim](https://github.com/fwupd/fwupd)
- [fwupd upstream: owner-key Secure Boot without Shim](https://github.com/fwupd/fwupd/discussions/9990)
- [Arch Linux: fwupd-efi package files](https://archlinux.org/packages/extra/any/fwupd-efi/files/)
- [sbctl manual](https://man.archlinux.org/man/sbctl.8)
- [systemd: logind.conf](https://man.archlinux.org/man/logind.conf.5)
- [Linux kernel: Platform profile selection](https://docs.kernel.org/userspace-api/sysfs-platform_profile.html)
- [libinput documentation](https://wayland.freedesktop.org/libinput/doc/latest/)

## Next step

Continue with chapter 07 to configure the core workstation services: complete
PipeWire routing, Bluetooth, removable media, the Secret Service, and optional
printing, then connect their user-session components without weakening the TTY
recovery path.
