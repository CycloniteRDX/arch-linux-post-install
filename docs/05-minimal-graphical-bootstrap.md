# 05 — Build the minimal graphical bootstrap

## Goal

Install the graphical and multimedia foundation, deploy one reviewed Niri
configuration from `niri-dotfiles`, and prove that a manual Niri session can
open Kitty and return safely to TTY.

This chapter makes the following changes:

- installs the userspace AMD graphics, Vulkan, and video-acceleration stack;
- installs Niri and Kitty from the official repositories;
- installs GTK and GNOME desktop portals;
- installs PipeWire, its PulseAudio and ALSA compatibility layers, and
  WirePlumber;
- installs RealtimeKit as the controlled real-time scheduling broker used by
  PipeWire and the desktop portal;
- installs polkit and the MATE graphical authentication agent;
- installs `xwayland-satellite` for X11 application compatibility;
- selects GNU Stow as the transparent dotfile deployment method;
- deploys the minimal `niri` package from `niri-dotfiles`;
- starts Niri manually from a TTY and verifies the session.

It does not enable a display manager or greeter, configure automatic login,
install a launcher or bar, add notifications, select a wallpaper, configure
locking or idle behavior, or deploy the final themed dotfiles.

## Prerequisites

- Chapters 01 through 04 are complete.
- The system is fully updated and NetworkManager has working internet access.
- Firewalld is enabled and the local TTY recovery path still works.
- `neon` has working `sudo` access.
- The public project repositories were cloned in chapter 02.
- The `niri-dotfiles` repository contains the reviewed chapter 05 bootstrap.

Run system-management commands as `neon` and use `sudo` only where shown.
Remain at a local TTY until the guide explicitly starts Niri.

## Resulting bootstrap

| Area | Canonical result |
| --- | --- |
| Kernel graphics driver | In-tree `amdgpu` |
| OpenGL userspace | Mesa |
| Vulkan driver | RADV from `vulkan-radeon` |
| Video acceleration | VA-API through `libva-mesa-driver` |
| Compositor | Niri from the official Arch repository |
| Terminal | Kitty |
| X11 compatibility | Automatic Niri integration with `xwayland-satellite` |
| Basic portal | `xdg-desktop-portal-gtk` |
| Screencast portal | `xdg-desktop-portal-gnome` |
| Audio server | PipeWire |
| Audio policy manager | WirePlumber |
| Real-time scheduling broker | RealtimeKit, activated through system D-Bus |
| Graphical authorization | polkit with `mate-polkit` |
| Dotfile deployment | GNU Stow |
| Session start | Manual `niri-session -l` from TTY |
| Greeter | Not installed or enabled |

The configuration intentionally contains only the session-critical polkit
autostart and two bindings:

| Binding | Action |
| --- | --- |
| `Super+Enter` | Open Kitty |
| `Super+Shift+E` | Show Niri's exit confirmation |

`Mod` means the Super key when Niri runs directly from a TTY.

## Confirm the AMD graphics path

Inspect the detected graphics controller and kernel driver:

```bash
lspci -k | grep -A 3 -E 'VGA|3D|Display'
ls -l /dev/dri
```

The integrated Radeon controller must report `Kernel driver in use: amdgpu`.
`/dev/dri` should contain at least one `card` device and one `renderD` device.
Stop if another kernel driver owns the AMD GPU or no DRM devices exist.

Do not install `xf86-video-amdgpu` for this Wayland bootstrap. Niri uses DRM,
GBM, and Mesa directly; the separate Xorg display driver is not required by
Niri or by `xwayland-satellite`.

## Audit conflicting multimedia packages

Check for audio servers or session managers that would overlap the canonical
PipeWire stack:

```bash
pacman -Q pulseaudio pulseaudio-bluetooth pipewire-media-session 2>&1
```

On the fresh runbook system, all three packages should be absent. If any is
installed, identify why before continuing. `pipewire-pulse` replaces
PulseAudio; WirePlumber replaces `pipewire-media-session`.

Also confirm that no greeter has been enabled prematurely:

```bash
systemctl is-enabled greetd.service 2>&1
systemctl is-active greetd.service 2>&1
```

`not-found`, `disabled`, and `inactive` are acceptable at this stage. A running
greeter is outside the chapter 05 recovery model.

## Install the graphical foundation

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then install the complete bootstrap in one upgrade transaction:

```bash
sudo pacman -Syu \
    mesa \
    vulkan-radeon \
    libva-mesa-driver \
    mesa-utils \
    vulkan-tools \
    libva-utils \
    niri \
    kitty \
    xwayland-satellite \
    xdg-desktop-portal \
    xdg-desktop-portal-gtk \
    xdg-desktop-portal-gnome \
    pipewire \
    pipewire-audio \
    pipewire-alsa \
    pipewire-pulse \
    wireplumber \
    rtkit \
    alsa-utils \
    polkit \
    mate-polkit \
    stow \
    ttf-dejavu
```

Read the complete transaction before accepting it. In current Arch packages,
`xdg-desktop-portal-gnome` depends on Nautilus because the GNOME portal uses it
for file selection. Accepting that dependency does not yet select Nautilus as
the project's final file manager; that application decision remains in
chapter 09.

The diagnostic packages are intentional:

| Package | Bootstrap check |
| --- | --- |
| `mesa-utils` | OpenGL renderer and XWayland check with `glxinfo` |
| `vulkan-tools` | Vulkan driver check with `vulkaninfo` |
| `libva-utils` | VA-API driver check with `vainfo` |
| `alsa-utils` | Device diagnostics and an audible ALSA compatibility test |
| `rtkit` | Controlled real-time scheduling for PipeWire and the Realtime portal |

`ttf-dejavu` supplies a dependable initial font for Kitty. The complete font
selection remains in chapter 08.

`gnome-keyring` and the Secret Service integration are intentionally deferred
to chapter 07, where their PAM and session lifecycle can be configured and
tested together. Their absence does not prevent this chapter's basic portal
and screencast foundation from starting.

Inspect package-manager messages and unresolved configuration files:

```bash
sudo pacdiff --output
```

Resolve any reported `.pacnew`, `.pacsave`, or `.pacorig` file before
continuing.

## Verify the installed package set

Confirm the explicit bootstrap packages:

```bash
pacman -Q \
    mesa \
    vulkan-radeon \
    libva-mesa-driver \
    mesa-utils \
    vulkan-tools \
    libva-utils \
    niri \
    kitty \
    xwayland-satellite \
    xdg-desktop-portal \
    xdg-desktop-portal-gtk \
    xdg-desktop-portal-gnome \
    pipewire \
    pipewire-audio \
    pipewire-alsa \
    pipewire-pulse \
    wireplumber \
    rtkit \
    alsa-utils \
    polkit \
    mate-polkit \
    stow \
    ttf-dejavu
```

Check the principal commands and the graphical polkit agent:

```bash
command -v niri niri-session kitty stow rtkitctl wpctl pw-play speaker-test vulkaninfo vainfo glxinfo
test -x /usr/lib/mate-polkit/polkit-mate-authentication-agent-1
```

Every command must resolve and the final test must exit successfully.

Do not enable `seatd.service`. The package is a Niri dependency, but this
systemd installation uses its existing logind session for device access.

## Keep user services activation-based

Do not run `sudo systemctl enable` for PipeWire, WirePlumber, the portals, or
`rtkit-daemon.service`. The first group consists of user-session services and
is started through user sockets, user D-Bus, and the graphical session target.
RealtimeKit is a system service activated on demand through system D-Bus; its
unit is static by design.

Inspect their installed state without changing it:

```bash
systemctl --user list-unit-files \
    'pipewire*' \
    'wireplumber*' \
    'xdg-desktop-portal*' \
    --no-pager
```

Static or indirectly activated units are normal. The meaningful functional
checks happen after `niri-session` has imported the graphical environment.

## Update the dotfiles repository

Move to the clone created in chapter 02 and obtain only a fast-forward update:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git pull --ff-only
```

The working tree must be clean before pulling. Stop if Git reports local
changes or a divergent branch; do not overwrite an unexplained configuration.

Confirm that the Stow package contains the reviewed bootstrap:

```bash
find niri -type f -print
sed -n '1,160p' niri/.config/niri/config.kdl
```

The only file should be `niri/.config/niri/config.kdl`. Review it before
creating any link into the home directory.

## Check the deployment target

Inspect whether a Niri configuration already exists:

```bash
ls -ld ~/.config/niri ~/.config/niri/config.kdl 2>/dev/null
```

No output is expected on a fresh installation. If the target file or a target
symlink already exists, stop and review it. Do not allow Stow to replace an
unknown configuration. Backups of earlier dotfiles should live outside the
active target path and must not be committed.

## Preview and deploy the Niri package

From the root of `niri-dotfiles`, simulate the operation first:

```bash
stow \
    --simulate \
    --verbose \
    --no-folding \
    --target="$HOME" \
    niri
```

The preview must propose links below `~/.config/niri` and report no conflict.
Then perform the same operation without `--simulate`:

```bash
stow \
    --verbose \
    --no-folding \
    --target="$HOME" \
    niri
```

Verify the deployed path:

```bash
ls -l ~/.config/niri/config.kdl
readlink -f ~/.config/niri/config.kdl
```

The resolved path must end in
`niri-dotfiles/niri/.config/niri/config.kdl`. The home-directory entry is a
symlink; the repository remains the source of truth.

## Validate the Niri configuration from TTY

Parse the deployed configuration before starting a compositor:

```bash
niri validate
```

The command must complete without a configuration error. Also confirm that
the config does not contain obsolete manual XWayland setup:

```bash
grep -nE 'xwayland-satellite|DISPLAY' ~/.config/niri/config.kdl
```

No output is expected. Current Niri integrates `xwayland-satellite`
automatically: it creates the X11 socket, exports `DISPLAY`, and launches the
satellite only when an X11 client connects. A manual `spawn-at-startup` or a
hard-coded display number would conflict with that integration.

The bootstrap also does not hard-code `us` or `es`. Niri can use the keyboard
settings provided by the system, while later host-specific dotfiles can record
the physical-layout differences between the two ThinkPads.

## Start the first Niri session manually

From the local TTY, run:

```bash
niri-session -l
```

The `-l` form is appropriate from the already authenticated TTY login shell
and avoids launching another login shell. More importantly, use
`niri-session`, not bare `niri`: the session helper imports the environment,
starts Niri through the systemd user manager, and activates the graphical
session target required by portals and other desktop services.

The screen should switch to Niri. No bar, launcher, notification daemon,
wallpaper, or lock screen is expected yet.

Press `Super+Enter`. Kitty must open. If it does not, return to a TTY using
`Ctrl+Alt+F2`, log in, and follow the recovery section below.

## Verify the session from Kitty

Confirm the compositor and session environment:

```bash
niri --version
printf '%s\n' \
    "XDG_SESSION_TYPE=$XDG_SESSION_TYPE" \
    "XDG_CURRENT_DESKTOP=$XDG_CURRENT_DESKTOP" \
    "WAYLAND_DISPLAY=$WAYLAND_DISPLAY" \
    "DISPLAY=$DISPLAY"
niri msg outputs
```

Expected results:

- `XDG_SESSION_TYPE=wayland`;
- `XDG_CURRENT_DESKTOP` identifies Niri;
- `WAYLAND_DISPLAY` is set;
- `DISPLAY` is set by Niri's XWayland integration;
- the internal panel appears in `niri msg outputs` with the expected mode and
  scale.

Confirm that Niri and the polkit authentication agent are running:

```bash
systemctl --user is-active niri.service
pgrep -a -f polkit-mate-authentication-agent-1
```

Niri must be active and exactly one MATE polkit agent should be visible. Test
the graphical authorization path with a command that makes no lasting change:

```bash
pkexec /usr/bin/true
printf 'pkexec exit status: %s\n' "$?"
```

A graphical password dialog should appear. After successful authentication,
the exit status must be `0`.

## Verify graphics and X11 compatibility

Check native Vulkan and VA-API support:

```bash
vulkaninfo --summary
vainfo
```

The Vulkan summary should identify the AMD RADV driver. `vainfo` should load
the Mesa VA-API driver and list supported decode or encode profiles. Warnings
about unsupported codecs are not equivalent to the driver failing to load.

Then run an X11 OpenGL client:

```bash
glxinfo -B
```

This command should trigger `xwayland-satellite` automatically and report the
AMD/Mesa renderer. Confirm the on-demand integration:

```bash
pgrep -a -f xwayland-satellite
journalctl --user -b -u niri.service --no-pager | tail -40
```

The Niri log should contain an X11 socket message. Do not add a manual
XWayland autostart merely because the process was absent before the first X11
client.

## Verify RealtimeKit, PipeWire, and WirePlumber

Ask the packaged control utility to start RealtimeKit if necessary, then read
one of its published properties directly through system D-Bus:

```bash
rtkitctl --start
busctl --system get-property \
    org.freedesktop.RealtimeKit1 \
    /org/freedesktop/RealtimeKit1 \
    org.freedesktop.RealtimeKit1 \
    MaxRealtimePriority
systemctl is-active rtkit-daemon.service
```

`rtkitctl --start` and the property query must succeed, the latter should print
an integer value, and the service should become active. Do not enable it: the
D-Bus request is supposed to activate the static unit on demand. RealtimeKit
grants bounded scheduling priority under policy; it does not turn the desktop
session into an unrestricted real-time process.

Inspect the active audio graph and its user services:

```bash
wpctl status
systemctl --user is-active \
    pipewire.service \
    pipewire-pulse.service \
    wireplumber.service
```

All three services should be active, and `wpctl status` should list the
ThinkPad audio devices. Set a conservative test volume and unmute the current
default output:

```bash
wpctl get-volume @DEFAULT_AUDIO_SINK@
wpctl set-volume @DEFAULT_AUDIO_SINK@ 35%
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
```

First test native PipeWire playback with the packaged ALSA voice sample:

```bash
pw-play /usr/share/sounds/alsa/Front_Center.wav
```

Then verify that ordinary ALSA clients also reach PipeWire through
`pipewire-alsa`:

```bash
speaker-test -D default -c 2 -t wav -l 1
```

The first command should play a centre voice. The second should announce the
front-left and front-right channels once and then exit. Hearing both tests
proves more than service activation: `pw-play` uses PipeWire natively, while
`speaker-test` exercises the ALSA compatibility path used by many programs.

If a command runs but remains silent, inspect the selected default sink in
`wpctl status`, its mute state, and the physical output before continuing. Do
not install PulseAudio or copy an unreviewed WirePlumber rule. Chapter 07 adds
`pavucontrol`, route and profile inspection, microphone recording, and
Bluetooth audio tests.

The minimal Niri file deployed in this chapter deliberately has no volume or
microphone media-key bindings. The ThinkPad mute keys and their LEDs are
therefore not a pass criterion here: chapter 06 verifies that the key events
and LED devices reach Linux, and chapter 10 deploys and tests the bindings.

Do not enable the PipeWire or WirePlumber units manually if they are working.
Their activation model is already providing the correct per-user lifecycle.

## Verify the portals

Start the portal broker for this validation without enabling it permanently:

```bash
systemctl --user start xdg-desktop-portal.service
systemctl --user is-active xdg-desktop-portal.service
systemctl --user --no-pager --full status \
    xdg-desktop-portal.service \
    xdg-desktop-portal-gtk.service \
    xdg-desktop-portal-gnome.service
```

The broker must be active. A backend can remain activation-based until a
client requests one of its interfaces, but no unit should be failed. Inspect
the logs if systemd reports a failure:

```bash
journalctl --user -b \
    -u xdg-desktop-portal.service \
    -u xdg-desktop-portal-gtk.service \
    -u xdg-desktop-portal-gnome.service \
    --no-pager
```

With `rtkit` installed, a new log message stating that
`org.freedesktop.RealtimeKit1` is not activatable is not expected. A message
recorded earlier in the current boot can remain in the journal after the
package is installed, so compare its timestamp with the latest portal start
instead of treating old journal history as a new failure.

Do not set `GDK_BACKEND` globally. Niri upstream explicitly warns that doing so
breaks the GNOME screencast portal.

Portal file selection and screen sharing receive application-level tests in
later chapters, after a browser and daily applications exist.

## Exit to TTY and verify the checkpoint

Press `Super+Shift+E`, accept Niri's confirmation dialog, and wait for the
original TTY shell to return.

Confirm that no greeter was enabled and that the ordinary system remains
healthy:

```bash
systemctl is-enabled greetd.service 2>&1
systemctl is-active greetd.service 2>&1
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

The greeter must remain absent or disabled. Investigate every failed unit; do
not hide failures merely to obtain an empty list.

Reboot once and repeat the manual launch:

```bash
sudo reboot
```

After logging in at TTY, run `niri validate`, start `niri-session -l`, open
Kitty, and exit again. This proves the bootstrap is reproducible across boots
without depending on stale user-service state.

## Recovery from a failed graphical start

If Niri shows a black screen, Kitty does not open, or input behaves
unexpectedly:

1. Press `Ctrl+Alt+F2` to reach another TTY.
2. Log in as `neon`.
3. Stop the user compositor service:

```bash
systemctl --user stop niri.service
```

4. Validate the configuration and inspect the current-boot log:

```bash
niri validate
journalctl --user -b -u niri.service --no-pager
```

5. Recheck the graphics driver and DRM devices:

```bash
lspci -k | grep -A 3 -E 'VGA|3D|Display'
ls -l /dev/dri
```

Do not enable a greeter while manual startup is unresolved.

## Remove the bootstrap links if necessary

To remove only the links owned by the `niri` Stow package:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow \
    --delete \
    --verbose \
    --target="$HOME" \
    niri
```

This removes deployed symlinks; it does not delete the tracked source file in
the repository. Check the result:

```bash
ls -ld ~/.config/niri ~/.config/niri/config.kdl 2>/dev/null
git status --short --branch
```

The repository must remain clean. Any untracked backup or generated file must
be reviewed before another deployment.

## Completion checklist

- [ ] The AMD controller uses the `amdgpu` kernel driver.
- [ ] Mesa, RADV, and the Mesa VA-API driver are installed.
- [ ] Niri, Kitty, both portal backends, PipeWire, WirePlumber, RealtimeKit,
      polkit, and `xwayland-satellite` are installed from official repositories.
- [ ] No overlapping PulseAudio server or obsolete PipeWire session manager is
      installed.
- [ ] The Niri config is deployed from `niri-dotfiles` with GNU Stow.
- [ ] `niri validate` succeeds from TTY.
- [ ] `niri-session -l` starts the graphical session manually.
- [ ] `Super+Enter` opens Kitty.
- [ ] `Super+Shift+E` exits Niri and returns to TTY.
- [ ] Niri exports the Wayland and X11 session variables.
- [ ] `xwayland-satellite` starts on demand without manual configuration.
- [ ] The graphical polkit prompt succeeds.
- [ ] RealtimeKit is available through system D-Bus without manual enabling.
- [ ] PipeWire and WirePlumber are active and `wpctl status` sees devices.
- [ ] Native PipeWire and ALSA-compatible speaker tests are audible.
- [ ] Portal services have no failed units.
- [ ] Vulkan, VA-API, and OpenGL identify the AMD/Mesa stack.
- [ ] No greeter or automatic graphical login has been enabled.
- [ ] The manual session still works after a reboot.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: AMDGPU](https://wiki.archlinux.org/title/AMDGPU)
- [ArchWiki: Hardware video acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)
- [ArchWiki: Vulkan](https://wiki.archlinux.org/title/Vulkan)
- [ArchWiki: Niri](https://wiki.archlinux.org/title/Niri)
- [ArchWiki: Kitty](https://wiki.archlinux.org/title/Kitty)
- [ArchWiki: PipeWire](https://wiki.archlinux.org/title/PipeWire)
- [ArchWiki: WirePlumber](https://wiki.archlinux.org/title/WirePlumber)
- [Arch Linux package: RealtimeKit](https://archlinux.org/packages/extra/x86_64/rtkit/)
- [Arch Linux package: ALSA utilities](https://archlinux.org/packages/extra/x86_64/alsa-utils/)
- [PipeWire: real-time module](https://pipewire.pages.freedesktop.org/pipewire/page_module_rt.html)
- [XDG Desktop Portal: Realtime interface](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Realtime.html)
- [ArchWiki: Polkit](https://wiki.archlinux.org/title/Polkit)
- [ArchWiki: XDG Desktop Portal](https://wiki.archlinux.org/title/XDG_Desktop_Portal)
- [Niri upstream: Getting Started](https://github.com/niri-wm/niri/wiki/Getting-Started)
- [Niri upstream: Important Software](https://github.com/niri-wm/niri/wiki/Important-Software)
- [Niri upstream: Xwayland](https://github.com/niri-wm/niri/wiki/Xwayland)
- [Niri upstream: Configuration Introduction](https://github.com/niri-wm/niri/wiki/Configuration%3A-Introduction)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next step

Continue with chapter 06 to configure ThinkPad firmware, suspend and lid
behavior, battery policy, thermals, hardware monitoring, function keys,
touchpad, and TrackPoint from the now-available TTY or Kitty environment.
