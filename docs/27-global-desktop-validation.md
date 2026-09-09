# Global desktop validation

## Purpose and completed state

This chapter closes the first complete RogueOS/Niri personalization cycle on
the first ThinkPad T14 Gen 1 AMD. It does not introduce another desktop layer.
It verifies the existing owners together, records two corrections discovered
by real use, and establishes the source state from which the first stable
`niri-dotfiles` release can be tagged.

The complete sequence passed hardware validation on 2026-09-09:

- GTK 3, GTK 4/libadwaita, Qt 6, icons, fonts, cursor and portals are visually
  coherent at scale 1.25;
- Mako resolves tested named icons through Papirus Dark and Papirus;
- Bluetooth starts with BlueZ available but the controller unpowered;
- Waybar can enable and disable Bluetooth across both BlueZ and rfkill states;
- Blueman Manager remains available on demand without permanent applet or tray
  processes;
- Bash, Nano, Micro, Vim, Kitty, Fuzzel, Mako, swaybg, swaylock, swayidle,
  Waybar, udiskie, PipeWire, WirePlumber, NetworkManager, firewalld, greetd and
  GNOME Keyring retain their intended ownership;
- lock, manual suspend/resume, monitor restoration, logout, tuigreet login and
  keyring unlock work in sequence;
- the normal signed UKI presents the RogueOS Plymouth path and TPM2 PIN, while
  the signed fallback remains the previously validated textual recovery path;
- system and user managers report no failed units.

The second ThinkPad still requires its own connector, mode, scale and external
display measurements. That remaining host-specific task does not prevent the
first target from becoming a stable release.

## Ownership boundary

This final state deliberately keeps one owner per role:

| Role | Owner | Final policy |
| --- | --- | --- |
| Compositor and session | Niri | One `niri --session` process |
| Status and Bluetooth action | Waybar | One bar; left click toggles, right click opens Blueman Manager |
| Notifications | Mako | Sole owner of `org.freedesktop.Notifications` |
| Wallpaper | swaybg | One static wallpaper; no rotation automation |
| Lock and idle | swaylock plus swayidle | Lock, monitor power, battery-only suspend and resume restoration |
| Audio graph | PipeWire plus WirePlumber | No parallel PulseAudio daemon |
| Bluetooth mechanism | BlueZ | Enabled/active daemon, unpowered controller at boot |
| Bluetooth graphical administration | Blueman Manager | On demand; no permanent applet or tray |
| Secrets | GNOME Keyring | Sole Secret Service owner, unlocked through PAM login |
| GTK appearance | GTK settings plus GSettings | Dark preference, Papirus Dark, Breeze cursor, Noto Sans |
| Qt 6 appearance | qt6ct plus Fusion | Midnight Circuit palette and portal dialogs |
| Boot presentation | Plymouth | Graphical normal UKI only; fallback remains textual |

Do not add another notification daemon, wallpaper renderer, power-profile
daemon, keyring provider, Bluetooth applet or idle coordinator merely to make a
status check pass.

## Start from the reviewed sources

Work only from clean, synchronized repositories. Do not copy a generated home
configuration back into Git without comparing it with the tracked source.

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
git fetch --prune --tags origin
git pull --ff-only
```

After the chapter 27 tags exist, a fresh reconstruction should select the
immutable checkpoint before deployment:

```bash
git switch --detach post-install-27-v1
git describe --tags --exact-match
```

## Correct Mako named-icon lookup

The chapter 23 configuration explicitly pointed Mako at Adwaita. During final
validation, an absolute Adwaita SVG path rendered correctly, proving that Mako
could display the image, but the tested Adwaita symbolic names did not resolve
through the installed directory layout. The equivalent Papirus names rendered
correctly and match the desktop-wide icon selection.

The tracked Mako block is therefore:

```ini
icons=1
icon-path=/usr/share/icons/Papirus-Dark:/usr/share/icons/Papirus
max-icon-size=48
icon-location=left
```

Both paths are intentional. Mako does not apply the complete desktop icon-theme
inheritance policy itself; listing the Papirus parent preserves broader name
coverage. Mako continues to add its hicolor and pixmaps fallbacks.

Deploy and reload the reviewed file:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --simulate --verbose --no-folding --target="$HOME" mako
stow --verbose --no-folding --target="$HOME" mako
makoctl reload
```

Test ordinary names rather than only absolute paths:

```bash
notify-send --icon=dialog-information \
    'Mako icon validation' 'Papirus information icon'
notify-send --icon=audio-volume-high \
    'Mako icon validation' 'Papirus audio icon'
notify-send --icon=network-wireless \
    'Mako icon validation' 'Papirus network icon'
```

All three icons were visible on the validated target.

## Make Bluetooth a genuinely on-demand surface

### Why `AutoEnable=false` is not the whole user-session policy

Chapter 07 keeps `/etc/bluetooth/main.conf` at:

```ini
[Policy]
AutoEnable=false
```

That controls BlueZ's controller-enable policy; it does not disable the system
service, suppress an XDG autostart entry, or provide a complete Waybar action.
The Arch Blueman package installs `/etc/xdg/autostart/blueman.desktop`. Starting
that applet permanently is unnecessary when Waybar already presents adapter
state and explicitly opens Blueman Manager for administration.

The dotfiles package therefore deploys the same basename at the higher-priority
user location:

```ini
[Desktop Entry]
Type=Application
Name=Blueman Applet
Hidden=true
```

This disables only automatic session startup. It does not remove the package,
edit `/etc`, prevent `blueman-manager` from running, or disable BlueZ.

### Why the old Waybar command was incomplete

On the target ThinkPad, powering Bluetooth down can leave the radio soft
blocked and temporarily remove the default controller from `bluetoothctl`.
The old inline action could issue `bluetoothctl power off`, but its matching
`bluetoothctl power on` then failed with `No default controller available`.

The executable `scripts/.local/bin/toggle-bluetooth` now:

1. powers down an available powered controller;
2. otherwise unblocks only Bluetooth through `rfkill`;
3. waits for the controller and firmware for up to three seconds;
4. requests `bluetoothctl power on`;
5. reports a bounded failure through the notification service.

Waybar calls that helper on left click and retains `blueman-manager` on right
click. Check syntax and Git mode before deployment:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
bash -n scripts/.local/bin/toggle-bluetooth
git ls-files --stage scripts/.local/bin/toggle-bluetooth
```

The mode must be `100755`. Preview and deploy both affected packages:

```bash
stow --simulate --verbose --no-folding --restow \
    --target="$HOME" autostart scripts waybar
stow --verbose --no-folding --restow \
    --target="$HOME" autostart scripts waybar
```

After a cold boot, before opening Blueman or clicking Waybar, validate:

```bash
pgrep -af 'blueman|bluetoothd'
rfkill list bluetooth
bluetoothctl show
systemctl is-enabled bluetooth.service
systemctl is-active bluetooth.service
```

The validated result contains `bluetoothd`, no `blueman-applet` or
`blueman-tray`, and an enumerated controller with `Powered: no`. The rfkill
soft block is clear, so the Waybar helper can act immediately. One left click
produces `Powered: yes`; a second returns to `Powered: no`. Right click opens
Blueman Manager only when graphical administration is requested.

## Close the GTK and Qt consistency review

Inventory the selected packages:

```bash
pacman -Q |
grep -E \
  '^(gtk3|gtk4|libadwaita|adw-gtk-theme|qt6ct|papirus-icon-theme|breeze|xdg-desktop-portal)( |-)'
```

Inspect the desktop preferences:

```bash
gsettings get org.gnome.desktop.interface color-scheme
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface icon-theme
gsettings get org.gnome.desktop.interface cursor-theme
gsettings get org.gnome.desktop.interface cursor-size
gsettings get org.gnome.desktop.interface font-name
gsettings get org.gnome.desktop.interface monospace-font-name
```

The validated target uses:

- `prefer-dark`;
- `adw-gtk3-dark` for GTK 3;
- Papirus Dark icons;
- `breeze_cursors` at 24 px;
- Noto Sans 10 and a compatible monospace face;
- GTK 4/libadwaita dark preference without copied GTK 4 CSS;
- qt6ct, Fusion, Midnight Circuit, Papirus Dark and portal-backed standard
  dialogs for Qt 6.

Confirm that no global override competes with those choices:

```bash
printf '%s\n' \
  "QT_QPA_PLATFORMTHEME=${QT_QPA_PLATFORMTHEME-}" \
  "QT_STYLE_OVERRIDE=${QT_STYLE_OVERRIDE-}" \
  "QT_QPA_PLATFORM=${QT_QPA_PLATFORM-}" \
  "GTK_THEME=${GTK_THEME-}" \
  "XCURSOR_THEME=${XCURSOR_THEME-}" \
  "XCURSOR_SIZE=${XCURSOR_SIZE-}"
```

`QT_QPA_PLATFORMTHEME=qt6ct`, `XCURSOR_THEME=breeze_cursors`, and
`XCURSOR_SIZE=24` are expected. `QT_STYLE_OVERRIDE`, `QT_QPA_PLATFORM`, and
`GTK_THEME` remain empty so applications retain supported toolkit and platform
selection.

Verify the portal owners:

```bash
systemctl --user --no-pager --full status \
    xdg-desktop-portal.service \
    xdg-desktop-portal-gnome.service \
    xdg-desktop-portal-gtk.service
systemctl --user --failed --no-pager
```

The one observed libadwaita warning about
`gtk-application-prefer-dark-theme` is informational for this configuration:
the applications follow the selected dark presentation and no portal unit
fails. Do not add unsupported GTK 4 CSS merely to silence that message.

Perform the visual review in real applications:

- Nautilus and another GTK 4/libadwaita application;
- a GTK 3 application;
- qt6ct and a Qt 6 application;
- Firefox and its portal-backed file picker;
- one XWayland client;
- selected, disabled, menu, tooltip and dialog states;
- pointer transitions between Niri, GTK, Qt, XWayland and swaylock.

The complete review passed at 1.25 scale without adding Qt 5, Kvantum, a forced
platform backend, or user GTK 4 CSS.

## Verify the running desktop

Check exclusive processes without relying on `pgrep -x` for names longer than
15 characters:

```bash
pgrep -af \
  '(^|/)(niri|waybar|mako|swaybg|swayidle|xwayland-satellite|gnome-keyring-daemon|udiskie)( |$)'
```

Niri, Waybar, Mako, swaybg, swayidle and udiskie must each have one intended
instance. GNOME Keyring must own the secret service. Xwayland-satellite may be
absent until an X11 client requests it; its earlier successful on-demand launch
is the relevant compatibility check.

```bash
busctl --user --no-pager list |
grep -E 'org.freedesktop.Notifications|org.freedesktop.secrets'
```

Mako must own notifications and GNOME Keyring must own secrets.

Verify audio, network and firewall state:

```bash
systemctl --user is-active \
    pipewire.service pipewire-pulse.service wireplumber.service
wpctl status
nmcli general status
nmcli device status
nmcli connection show --active
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

The validated mobile hotspot correctly appeared as metered. The public zone
owned the Wi-Fi interface, exposed no custom port, and retained only the
expected DHCPv6 client service.

Confirm clipboard and editor configuration:

```bash
printf 'RogueOS clipboard validation\n' | wl-copy
wl-paste
bash --noprofile --norc -n ~/.bash_profile
bash --noprofile --norc -n ~/.bashrc
jq empty ~/.config/micro/settings.json
vim -Nu ~/.config/vim/vimrc -n -es -c 'qa!'
test -d ~/.local/state/vim/undo
test -d ~/.local/state/vim/swap
```

Nano, Micro and Vim also passed their real visual, syntax and clipboard tests.

## Verify lifecycle and boot boundaries

Perform these disruptive checks only after saving work:

1. lock through the configured action and unlock;
2. run `systemctl suspend`, then verify swaylock and monitor restoration;
3. log out, verify tuigreet privacy and presentation, then log in again;
4. confirm the keyring and notification bus owners;
5. power off completely and boot the normal UKI;
6. verify firmware, systemd-boot, RogueOS Plymouth, TPM2 PIN, tuigreet and Niri;
7. confirm that Bluetooth remains unpowered before its first Waybar action.

After each session transition:

```bash
systemctl --failed --no-pager
systemctl --user --failed --no-pager
```

Both lists remained empty. The previous chapter 26 validation already proved
the deliberately textual fallback UKI and manual LUKS passphrase path; do not
repeat recovery boot merely to accumulate duplicate evidence.

## Prove clean Stow reconstruction

From a clean repository, preview every selected package together:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --simulate --verbose --no-folding --target="$HOME" \
    niri autostart mimeapps waybar fuzzel mako wallpapers \
    swaylock kitty theme qt6ct scripts bash nano micro vim
```

The preview must report no conflict. In particular, the active Mako file must
be the Stow link, not an untracked regular copy:

```bash
readlink -f ~/.config/mako/config
cmp mako/.config/mako/config ~/.config/mako/config
```

Finish with repository and syntax checks:

```bash
git status --short --branch
git diff --check
bash -n scripts/.local/bin/idle-suspend
bash -n scripts/.local/bin/toggle-bluetooth
jq empty waybar/.config/waybar/config.jsonc
jq empty micro/.config/micro/settings.json
niri validate
```

## Classification and release checkpoint

The first target is now classified **READY — FIRST PERSONALIZED DESKTOP
RELEASE**. Deferred second-ThinkPad output measurements and optional future
component experiments do not weaken this result.

After committing the matching documentation in both repositories:

1. create `post-install-27-v1` in `arch-linux-post-install`;
2. create the same `post-install-27-v1` checkpoint in `niri-dotfiles`;
3. create `v1.0.0` in `niri-dotfiles` at that same reviewed commit;
4. push the annotated tags explicitly;
5. do **not** create another `v1.0.0` in this post-install repository, whose
   baseline release already uses that name.

The numbered tag preserves the chapter relationship. The semantic dotfiles tag
is the ordinary complete-desktop reconstruction target.

## Sources

- [BlueZ sample `main.conf`](https://github.com/bluez/bluez/blob/master/src/main.conf)
- [Blueman PowerManager](https://github.com/blueman-project/blueman/blob/main/blueman/plugins/applet/PowerManager.py)
- [Desktop Application Autostart Specification](https://specifications.freedesktop.org/autostart-spec/latest/)
- [Mako configuration manual](https://man.archlinux.org/man/mako.5.en)
- [GTK configuration](https://wiki.archlinux.org/title/GTK)
- [Qt configuration](https://wiki.archlinux.org/title/Qt)
- [XDG Desktop Portal documentation](https://flatpak.github.io/xdg-desktop-portal/docs/)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next step

Commit and tag the reviewed release state. After that, ordinary work moves from
first-pass personalization to maintenance, measured second-host integration,
and optional one-component experiments with explicit rollback.
