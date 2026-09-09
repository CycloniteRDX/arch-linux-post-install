# 28 — Reconstruct the complete workstation

## Goal

Reconstruct the hardware-validated RogueOS/Niri workstation from a completed
`arch-linux-runbook` installation without replaying every explanatory
chapter. This is the concise deployment path for the first ThinkPad profile.

Use chapters 00 through 27 and the handbook when a command, ownership boundary,
or failure needs explanation. Do not use this chapter to repair an unknown or
partially configured system.

## Fixed release inputs

This reconstruction uses:

- `post-install-28-v1` from this repository;
- `v1.0.0` from `niri-dotfiles`;
- the first ThinkPad's measured `eDP-1` policy;
- the `neon` account and `~/Projects/CycloniteRDX` layout.

`v1.0.0` is immutable. Chapter 28 adds deployment material outside the
dotfiles release and does not require `v1.0.1`.

## Confirm the runbook handoff

```bash
findmnt /
findmnt /home
findmnt /boot
swapon --show
systemctl is-enabled NetworkManager.service
systemctl is-enabled systemd-timesyncd.service
sudo sbctl status
sudo sbctl verify
sudo bootctl --esp-path=/boot list
systemctl --failed --no-pager
```

Stop unless the encrypted LVM layout, ext4 filesystems, root-only ESP mount,
signed normal and fallback UKIs, Secure Boot, network, time synchronization,
disk swap and TTY recovery path match chapter 00.

## Obtain the frozen post-install sources

```bash
project_root="$HOME/Projects/CycloniteRDX"
postinstall_repo="$project_root/arch-linux-post-install"

mkdir -p "$project_root"
git clone \
  https://github.com/CycloniteRDX/arch-linux-post-install.git \
  "$postinstall_repo"

git -C "$postinstall_repo" fetch --prune --tags origin
git -C "$postinstall_repo" switch --detach post-install-28-v1
git -C "$postinstall_repo" describe --tags --exact-match
git -C "$postinstall_repo" status --short
```

The description must be `post-install-28-v1` and the final status must
print nothing.

## Install the complete package manifest

```bash
sed -n '1,240p' \
  "$postinstall_repo/assets/packages/rogueos-workstation.txt"

mapfile -t workstation_packages < <(
  sed -E '/^[[:space:]]*(#|$)/d' \
    "$postinstall_repo/assets/packages/rogueos-workstation.txt"
)

test "${#workstation_packages[@]}" -gt 0
printf '%s\n' "${workstation_packages[@]}"
sudo pacman -Syu --needed "${workstation_packages[@]}"
sudo pacdiff --output
```

Resolve every reported package configuration file before continuing.

The manifest excludes these conditional additions:

```bash
# Optional CJK coverage
sudo pacman -Syu --needed noto-fonts-cjk

# Optional Python project tooling
sudo pacman -Syu --needed python-pip python-pipx

# Optional printing stack
sudo pacman -Syu --needed \
  cups cups-filters cups-pk-helper system-config-printer

# Optional mirror automation
sudo pacman -Syu --needed reflector
```

Run only the required blocks. Paru remains the separately reviewed chapter 16
procedure and is not part of the official-package manifest.

## Deploy project-owned system files

```bash
sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/systemd/zram-generator.conf" \
  /etc/systemd/zram-generator.conf

sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/systemd/logind.conf.d/70-thinkpad-suspend.conf" \
  /etc/systemd/logind.conf.d/70-thinkpad-suspend.conf

sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/tlp/10-thinkpad-battery.conf" \
  /etc/tlp.d/10-thinkpad-battery.conf

sudo diff -u \
  "$postinstall_repo/assets/systemd/zram-generator.conf" \
  /etc/systemd/zram-generator.conf
sudo diff -u \
  "$postinstall_repo/assets/systemd/logind.conf.d/70-thinkpad-suspend.conf" \
  /etc/systemd/logind.conf.d/70-thinkpad-suspend.conf
sudo diff -u \
  "$postinstall_repo/assets/tlp/10-thinkpad-battery.conf" \
  /etc/tlp.d/10-thinkpad-battery.conf
```

The three comparisons must print nothing. Reboot later instead of restarting
`systemd-logind` inside the graphical session.

## Activate maintenance, firewall, firmware and power

```bash
sudo systemctl enable --now paccache.timer
sudo systemctl enable --now fstrim.timer

sudo systemctl enable --now firewalld.service
sudo firewall-cmd --set-default-zone=public
if sudo firewall-cmd --permanent --zone=public --query-service=ssh; then
  sudo firewall-cmd --permanent --zone=public --remove-service=ssh
fi
if sudo firewall-cmd --permanent --zone=public --query-forward; then
  sudo firewall-cmd --permanent --zone=public --remove-forward
fi
sudo firewall-cmd --check-config
sudo firewall-cmd --reload

sudo cp --archive \
  /etc/fwupd/fwupd.conf \
  /etc/fwupd/fwupd.conf.before-rogueos
sudo fwupdmgr modify-config P2pPolicy nothing
sudo systemctl mask --now passim.service
sudo systemctl enable --now fwupd-refresh.timer
sudo sbctl sign --save \
  --output /usr/lib/fwupd/efi/fwupdx64.efi.signed \
  /usr/lib/fwupd/efi/fwupdx64.efi
```

Edit the package-owned fwupd file:

```bash
sudoedit /etc/fwupd/fwupd.conf
```

Preserve its contents and ensure these settings exist in their indicated
sections:

```ini
[fwupd]
P2pPolicy=nothing

[uefi_capsule]
DisableShimForSecureBoot=true
```

```bash
sudo systemctl restart fwupd.service
sudo systemctl enable --now tlp.service
sudo systemctl enable --now tlp-pd.service
sudo tlp start

sudo tlp-stat -s
sudo tlp-stat -b
tlpctl list
tlpctl get
```

The battery report must confirm BAT0 threshold support before retaining the
installed 75/80 policy.

## Configure Bluetooth

```bash
sudo systemctl enable --now bluetooth.service
grep -nE '^\[Policy\]|^#?AutoEnable=' /etc/bluetooth/main.conf
sudo cp --archive \
  /etc/bluetooth/main.conf \
  /etc/bluetooth/main.conf.pre-autoenable
sudoedit /etc/bluetooth/main.conf
```

Set the existing `[Policy]` entry to:

```ini
AutoEnable=false
```

```bash
grep -nE '^\[Policy\]|^AutoEnable=' /etc/bluetooth/main.conf
bluetoothctl power off || true
systemctl is-enabled bluetooth.service
systemctl is-active bluetooth.service
```

Do not create a duplicate `[Policy]` section. The dotfiles deployment
below suppresses Blueman's permanent applet and gives Waybar the rfkill-aware
toggle.

## Create the user environment

Run as `neon`:

```bash
xdg-user-dirs-update
xdg-user-dir DOWNLOAD
xdg-user-dir DOCUMENTS
xdg-user-dir PICTURES

fc-match sans-serif
fc-match monospace
fc-match 'Noto Color Emoji'
```

## Integrate GNOME Keyring with PAM

```bash
sudo cp --archive /etc/pam.d/login \
  /etc/pam.d/login.before-gnome-keyring
sudo cp --archive /etc/pam.d/passwd \
  /etc/pam.d/passwd.before-gnome-keyring
sudo cp --archive /etc/pam.d/greetd \
  /etc/pam.d/greetd.before-gnome-keyring

sudoedit /etc/pam.d/login
sudoedit /etc/pam.d/passwd
sudoedit /etc/pam.d/greetd
```

Add each line only once and preserve every packaged entry:

```pam
# /etc/pam.d/login and /etc/pam.d/greetd, after auth entries
auth       optional     pam_gnome_keyring.so

# /etc/pam.d/login and /etc/pam.d/greetd, after session entries
session    optional     pam_gnome_keyring.so auto_start

# /etc/pam.d/passwd
password   optional     pam_gnome_keyring.so
```

```bash
sudo grep -n 'pam_gnome_keyring' \
  /etc/pam.d/login \
  /etc/pam.d/passwd \
  /etc/pam.d/greetd
test -e /usr/lib/security/pam_gnome_keyring.so
```

Authenticate on a second TTY before ending the original shell. Restore the
three backups from the original TTY if authentication fails.

## Prepare the two UKIs

```bash
boot_backup=/var/lib/rogueos-backups/boot-before-chapter-28
sudo test ! -e "$boot_backup"
sudo install -d -o root -g root -m 0700 "$boot_backup"
sudo cp --archive \
  /etc/kernel/cmdline \
  /etc/mkinitcpio.conf \
  /etc/mkinitcpio.d/linux.preset \
  "$boot_backup/"

luks_device=/dev/nvme0n1p2
luks_uuid="$(sudo cryptsetup luksUUID "$luks_device")"
test -n "$luks_uuid"
printf '%s\n' "$luks_uuid"

printf '%s\n' \
  "rd.luks.name=$luks_uuid=cryptlvm rd.luks.options=$luks_uuid=discard,tpm2-device=auto zswap.enabled=0 root=/dev/mapper/vg0-root rw quiet splash" |
  sudo tee /etc/kernel/cmdline

printf '%s\n' \
  "rd.luks.name=$luks_uuid=cryptlvm rd.luks.options=$luks_uuid=discard zswap.enabled=0 root=/dev/mapper/vg0-root rw" |
  sudo tee /etc/kernel/cmdline-fallback
```

Edit only the active `HOOKS=` line:

```bash
sudoedit /etc/mkinitcpio.conf
```

Use:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole plymouth block sd-encrypt lvm2 filesystems fsck)
```

Install the preset and PCR policy:

```bash
sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/mkinitcpio/linux.preset" \
  /etc/mkinitcpio.d/linux.preset

sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/kernel/uki.conf" \
  /etc/kernel/uki.conf
```

Generate a PCR-policy pair only when neither key exists:

```bash
pcr_private=/etc/systemd/tpm2-pcr-private-key-initrd.pem
pcr_public=/etc/systemd/tpm2-pcr-public-key-initrd.pem

if sudo test -e "$pcr_private" || sudo test -e "$pcr_public"; then
  sudo test -s "$pcr_private"
  sudo test -s "$pcr_public"
else
  sudo ukify genkey --config=/etc/kernel/uki.conf
fi

sudo chown root:root \
  /etc/kernel/uki.conf "$pcr_private" "$pcr_public"
sudo chmod 0644 /etc/kernel/uki.conf "$pcr_public"
sudo chmod 0600 "$pcr_private"
```

Do not regenerate a missing half of an existing pair. Restore its encrypted
backup or diagnose the discrepancy.

Install Plymouth and build both UKIs:

```bash
sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/plymouth/rogueos/rogueos.plymouth" \
  /usr/share/plymouth/themes/rogueos/rogueos.plymouth

sudo plymouth-set-default-theme rogueos
plymouth-set-default-theme
sudo mkinitcpio -P

sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo lsinitcpio /boot/EFI/Linux/arch-linux.efi |
  grep -E 'plymouth|themes/(rogueos|spinner)/'
sudo lsinitcpio /boot/EFI/Linux/arch-linux-fallback.efi |
  grep -E 'plymouth|themes/(rogueos|spinner)/' || true
sudo sbctl verify
```

Normal must include TPM2, `quiet splash` and Plymouth. Fallback must
contain none of them and must remain signed.

## Enroll TPM2 after proving recovery

Do not reduce credential enrollment to an unattended command. Complete:

1. chapter 12's encrypted recovery media and restore drill;
2. chapter 20's tested LUKS passphrase and recovery key;
3. its before-enrollment LUKS header and PCR-key backups;
4. normal and fallback boot tests without a TPM token.

Then follow
[chapter 20's enrollment command](20-tpm2-bound-luks-unlock.md#enroll-one-tpm2-token-last)
exactly. Each ThinkPad needs its own PIN, recovery key, LUKS header backup and
PCR private key. None belongs in Git.

```bash
sudo systemd-cryptenroll "$luks_device"
sudo cryptsetup luksDump "$luks_device"
sudo cryptsetup open --test-passphrase "$luks_device"
sudo sbctl verify
```

## Install the frozen dotfiles release

```bash
dotfiles_repo="$project_root/niri-dotfiles"

git clone \
  https://github.com/CycloniteRDX/niri-dotfiles.git \
  "$dotfiles_repo"

git -C "$dotfiles_repo" fetch --prune --tags origin
git -C "$dotfiles_repo" switch --detach v1.0.0
git -C "$dotfiles_repo" describe --tags --exact-match
git -C "$dotfiles_repo" status --short

install -d -m 0700 \
  "$HOME/.local/state/vim/undo" \
  "$HOME/.local/state/vim/swap"
```

The description must be `v1.0.0` and the status must print nothing.

Preview every package together:

```bash
cd "$dotfiles_repo"

stow --simulate --verbose --no-folding --target="$HOME" \
  niri autostart mimeapps waybar fuzzel mako wallpapers \
  swaylock kitty theme qt6ct scripts bash nano micro vim
```

Stop on any conflict. Back up and compare an existing target; never use
`--adopt` as a shortcut. If the preview is clean:

```bash
stow --verbose --no-folding --target="$HOME" \
  niri autostart mimeapps waybar fuzzel mako wallpapers \
  swaylock kitty theme qt6ct scripts bash nano micro vim

systemctl --user daemon-reload
```

Apply the settings also stored in the desktop settings database:

```bash
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
gsettings set org.gnome.desktop.interface gtk-theme 'adw-gtk3-dark'
gsettings set org.gnome.desktop.interface icon-theme 'Papirus-Dark'
gsettings set org.gnome.desktop.interface cursor-theme 'breeze_cursors'
gsettings set org.gnome.desktop.interface cursor-size 24
gsettings set org.gnome.desktop.interface font-name 'Noto Sans 10'

systemctl --user enable --now evolution-alarm-notify.service
```

Check the absolute qt6ct palette path before starting Niri:

```bash
grep '^color_scheme_path=' "$HOME/.config/qt6ct/qt6ct.conf"
test "$USER" = neon
```

The released file targets `/home/neon`. Adapt it in a later host-specific
commit instead of modifying the immutable tag.

## Validate Niri before enabling greetd

From TTY1:

```bash
niri validate
niri-session
```

Inside Niri, verify Kitty, Fuzzel, Waybar, Mako, swaybg, swayidle, lock,
notifications, audio, brightness, network, Bluetooth and logout. Return to TTY
before enabling graphical login.

## Install and enable graphical login

```bash
sudo cp --archive \
  /etc/greetd/config.toml \
  /etc/greetd/config.toml.before-rogueos

sudo install -D -o root -g root -m 0644 \
  "$postinstall_repo/assets/greetd/config.toml" \
  /etc/greetd/config.toml

sudo diff -u \
  "$postinstall_repo/assets/greetd/config.toml" \
  /etc/greetd/config.toml

if sudo test -d /var/cache/tuigreet; then
  sudo find /var/cache/tuigreet -maxdepth 1 -type f \
    -name 'lastuser*' -delete
fi
```

Keep an authenticated TTY3 open, end the manual Niri session, then:

```bash
sudo systemctl enable greetd.service
sudo systemctl start greetd.service
```

Log in through tuigreet and verify:

```bash
loginctl session-status
systemctl is-enabled greetd.service
systemctl is-active greetd.service
busctl --user --no-pager list |
  grep -E 'org.freedesktop.Notifications|org.freedesktop.secrets'
```

## Compact final validation

```bash
systemctl --failed --no-pager
systemctl --user --failed --no-pager

systemctl is-enabled \
  NetworkManager.service \
  systemd-timesyncd.service \
  firewalld.service \
  bluetooth.service \
  tlp.service \
  tlp-pd.service \
  greetd.service

systemctl --user is-active \
  pipewire.service \
  pipewire-pulse.service \
  wireplumber.service

niri validate
git -C "$dotfiles_repo" status --short
sudo firewall-cmd --zone=public --list-all
sudo tlp-stat -s
swapon --show
sudo sbctl verify
sudo bootctl --esp-path=/boot list
```

Power off completely and test:

1. normal UKI, RogueOS Plymouth and TPM2 PIN;
2. tuigreet, Niri and GNOME Keyring unlock;
3. Bluetooth unpowered before its first Waybar action;
4. lock, suspend, resume and monitor restoration;
5. textual fallback UKI and manual LUKS recovery.

After the cold boot:

```bash
cat /proc/cmdline
systemctl --failed --no-pager
systemctl --user --failed --no-pager
rfkill list bluetooth
bluetoothctl show
pgrep -af \
  '(^|/)(niri|waybar|mako|swaybg|swayidle|xwayland-satellite|gnome-keyring-daemon|udiskie)( |$)'
```

## Completion and checkpoint

The reconstruction is complete only when:

- the package manifest installs without a partial upgrade;
- project-owned system files match their sources;
- package-owned configuration and PAM changes are reviewed;
- the dotfiles clone remains detached at clean `v1.0.0`;
- normal and fallback retain their distinct policies;
- manual recovery works independently of TPM2;
- the chapter 27 global validation still passes.

After a clean-install test, create `post-install-28-v1` only in this
repository. Do not move or recreate `v1.0.0`, and do not create a
matching dotfiles or handbook tag.

## Detailed references

- [Package and maintenance baseline](02-package-and-maintenance.md)
- [Storage, zram, discard and TRIM](03-storage-and-memory.md)
- [Firewalld baseline](04-security-and-network.md)
- [ThinkPad hardware and power](06-thinkpad-hardware-and-power.md)
- [Bluetooth, removable media and GNOME Keyring](07-core-workstation-services.md)
- [Login, lock and idle lifecycle](11-login-lock-and-idle.md)
- [Backup and recovery](12-backup-and-recovery.md)
- [Plymouth graphical boot](19-plymouth-graphical-boot.md)
- [TPM2-bound LUKS unlock](20-tpm2-bound-luks-unlock.md)
- [RogueOS Plymouth theme](26-plymouth-visual-refinement.md)
- [Global desktop validation](27-global-desktop-validation.md)
