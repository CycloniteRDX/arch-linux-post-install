# 26 — Refine Plymouth with a minimal RogueOS theme

## Goal

Replace the packaged BGRT presentation inside the normal UKI with a small,
project-owned Plymouth descriptor that uses the Midnight Circuit background,
Arch's packaged watermark, and the packaged encrypted-root input controls.
Keep the fallback UKI textual and independent of Plymouth.

The resulting ownership model is:

| Item | Owner |
| --- | --- |
| Theme descriptor | This repository |
| Spinner, password-entry controls, and Arch watermark | Arch `plymouth` package |
| Installed system theme | `/usr/share/plymouth/themes/rogueos` |
| Theme selection | `/etc/plymouth/plymouthd.conf` |
| Normal and fallback UKIs | mkinitcpio, ukify, and sbctl |

This chapter does not replace the firmware logo, add a UKI `.splash` image,
edit package-owned themes, introduce a script plugin, delay boot for an
animation, or change LUKS, TPM2, greetd, PAM, or Niri.

## Status and prerequisites

This refinement passed cold-boot hardware validation on the target ThinkPad
T14 Gen 1 AMD on 2026-09-09. The normal UKI displayed the dark RogueOS
presentation, Arch watermark, and functional TPM PIN field; a rejected PIN
cleared the protected input and allowed a successful retry. Boot then reached
tuigreet with no failed units or Plymouth/cryptsetup errors.

The fallback UKI remained verbose and contained no Plymouth files. On the
current systemd stack it still discovered the enrolled LUKS2 TPM2 token and
requested its PIN; this is compatible with a Plymouth-independent recovery
image and is explained below. The strong LUKS passphrase was separately
verified with `cryptsetup --test-passphrase`.

Before applying this chapter:

- chapters 00 through 25 are complete;
- normal TPM2-plus-PIN boot and textual fallback boot both work;
- the strong LUKS passphrase and recovery material are available;
- `/boot` is the mounted ESP;
- Secure Boot is enabled and both UKIs are signed;
- the current working tree is clean and synchronized.

Do not reboot after a failed install, UKI build, or signature check.

## Why this is not a modified BGRT theme

BGRT is firmware data. The ThinkPad firmware exposes its Lenovo bitmap before
Linux owns the display, and Plymouth's packaged `bgrt` theme may retain that
firmware background. Replacing files under the packaged BGRT directory would
not reliably replace the firmware phase and would be overwritten by package
updates.

The RogueOS theme instead uses the packaged `two-step` module without
`UseFirmwareBackground=true`. The expected sequence is therefore:

1. Lenovo firmware image;
2. systemd-boot;
3. Plymouth's dark background and Arch watermark;
4. tuigreet after the encrypted root and userspace boot complete.

The Arch watermark is the package-owned
`/usr/share/plymouth/themes/spinner/watermark.png`. The project owns only the
descriptor that positions and colours the packaged components.

## Audit the current state

Run from any directory because these commands use absolute system paths:

```bash
pacman -Q plymouth mkinitcpio systemd systemd-ukify sbctl cryptsetup
plymouth-set-default-theme
plymouth-set-default-theme -l
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
grep -E '^(fallback_cmdline|fallback_options)=' /etc/mkinitcpio.d/linux.preset
sudo sbctl verify
systemctl --failed --no-pager
```

The established design must still show Plymouth in the common hook list, a
normal command line with `quiet splash`, and a fallback preset that skips
`autodetect,plymouth` and uses its separate command-line source.

Inspect the package-owned assets before depending on them:

```bash
file /usr/share/plymouth/themes/spinner/watermark.png
find /usr/share/plymouth/themes/spinner -maxdepth 1 -type f \
  \( -name 'entry.png' -o -name 'lock.png' -o -name 'bullet.png' \
     -o -name 'capslock.png' -o -name 'keyboard.png' \
     -o -name 'animation-*.png' -o -name 'throbber-*.png' \) \
  -printf '%f\n' | sort
```

Stop if the installed `plymouth` package no longer supplies the referenced
directory or encrypted-root controls.

## Enter the repository explicitly

Commands using `assets/...` are relative to the repository. Enter it first
and confirm the tracked source exists:

```bash
cd "$HOME/Projects/CycloniteRDX/arch-linux-post-install"
git status --short --branch
test -f assets/plymouth/rogueos/rogueos.plymouth
```

If `test` fails, update the repository before copying anything:

```bash
git pull --ff-only
test -f assets/plymouth/rogueos/rogueos.plymouth
```

`--ff-only` stops instead of creating an accidental merge. Do not continue
from another directory, because `install` would then look for an unrelated
`assets/` path.

## Preserve an existing custom experiment

An existing project-owned experiment is not package state. Move it outside
the active theme directory before installing the reviewed version. First pick
a new, non-existing backup destination:

```bash
backup_dir=/var/lib/rogueos-backups/plymouth/rogueos-before-chapter-26
sudo test ! -e "$backup_dir"
```

Continue only if the final command exits successfully. If the active custom
directory exists, preserve it:

```bash
if sudo test -d /usr/share/plymouth/themes/rogueos; then
  sudo install -d -o root -g root -m 0700 \
    /var/lib/rogueos-backups/plymouth
  sudo mv -- /usr/share/plymouth/themes/rogueos "$backup_dir"
fi
```

Keep backup directories outside `/usr/share/plymouth/themes`. A copied
`.plymouth` descriptor left inside that search tree can appear as a duplicate
theme and makes the active inventory ambiguous.

## Install and select the reviewed descriptor

Still inside the repository:

```bash
sudo install -D -o root -g root -m 0644 \
  assets/plymouth/rogueos/rogueos.plymouth \
  /usr/share/plymouth/themes/rogueos/rogueos.plymouth

sudo diff -u \
  assets/plymouth/rogueos/rogueos.plymouth \
  /usr/share/plymouth/themes/rogueos/rogueos.plymouth

sudo stat -c '%U:%G %a %n' \
  /usr/share/plymouth/themes/rogueos \
  /usr/share/plymouth/themes/rogueos/rogueos.plymouth
```

`diff` must print nothing. The directory and file should be `root:root` with
modes `755` and `644`. The theme deliberately contains no copied PNG or
`.script` file.

Select it without using Plymouth's implicit rebuild shortcut:

```bash
sudo plymouth-set-default-theme rogueos
plymouth-set-default-theme
sudo sed -n '1,80p' /etc/plymouth/plymouthd.conf
```

The selected theme must be `rogueos`. The project uses an explicit
`mkinitcpio -P` so both named UKI builds and sbctl post-hook results remain
visible.

## Rebuild and inspect both UKIs

```bash
sudo mkinitcpio -P
sudo bootctl list
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

The normal build must run the Plymouth hook, build the UKI, and sign it. The
fallback build must explicitly skip `autodetect,plymouth`, then build and sign
its own UKI. Missing optional firmware warnings already classified elsewhere
do not equal a build failure.

Inspect theme inclusion:

```bash
sudo lsinitcpio /boot/EFI/Linux/arch-linux.efi |
grep -E 'etc/plymouth/plymouthd.conf|themes/(rogueos|spinner)/'

sudo lsinitcpio /boot/EFI/Linux/arch-linux-fallback.efi |
grep -E 'plymouth|themes/(rogueos|spinner)' || true
```

Normal must include the selected descriptor and its referenced Spinner
assets. The fallback command must print nothing.

`sbctl verify` may report the raw `/boot/vmlinuz-linux` as unsigned. That file
is an input to the signed UKIs, not the EFI artifact executed by systemd-boot;
both UKIs and all executed bootloader copies must show valid signatures.

## Validate the normal boot

Cold boot the default `arch-linux.efi` entry and verify:

1. the firmware Lenovo image and systemd-boot remain recognizable phases;
2. Plymouth changes to the dark Midnight Circuit background;
3. the Arch watermark is visible above the encrypted-root control;
4. the lock, protected input, and input dots are readable;
5. the valid TPM PIN unlocks root and boot reaches tuigreet;
6. no unexpected console flash, hang, or theme asset is shown.

Do not intentionally repeat wrong TPM PIN attempts: they contribute to TPM
dictionary-attack protection. The one observed rejected attempt established
that the input resets cleanly; ordinary validation uses the correct PIN.

After login:

```bash
cat /proc/cmdline
systemctl --failed --no-pager
journalctl -b --no-pager |
grep -Ei 'plymouth.*(error|failed)|systemd-cryptsetup.*(error|failed)' || true
```

The normal command line must contain `tpm2-device=auto quiet splash`, failed
units must be zero, and the filtered journal should be empty.

### Why the spinner may not be visible

The `two-step` module uses animation during ordinary progress but replaces it
with the password dialog while a systemd password request is active. After
unlock, a fast SSD may reach the greeter before the spinner becomes
perceptible. A working input control and clean transition are stronger evidence
than artificially delaying boot to display an animation.

### Why a wrong PIN may only clear the dots

Systemd-cryptsetup records the rejection, but this theme may reset the
protected entry without rendering the journal text. A cleared input plus a
successful retry is acceptable. Use the journal for the exact diagnostic;
do not add fake status text or expose the secret type merely for decoration.

## Validate the textual fallback

Select `arch-linux-fallback.efi` once from systemd-boot. Confirm that it shows
ordinary early-boot text, contains no Plymouth presentation, reaches the
installed system, and reports zero failed units.

After login:

```bash
cat /proc/cmdline
systemctl --failed --no-pager
sudo lsinitcpio /boot/EFI/Linux/arch-linux-fallback.efi |
grep -E 'plymouth|themes/(rogueos|spinner)' || true
```

The running command line must omit `quiet`, `splash`, and
`tpm2-device=auto`; the initramfs query must print nothing.

### TPM2 token discovery is separate from Plymouth

On the validated systemd version, fallback still offered the TPM PIN. The
enrolled `systemd-tpm2` token is stored in the LUKS2 metadata and current
systemd-cryptsetup can discover suitable tokens automatically. Omitting the
explicit command-line option therefore does not guarantee that the token will
be ignored.

This does not invalidate the fallback. It remains independent of Plymouth,
quiet output, and the host-pruned initramfs. If TPM authorization is
unavailable, the preserved password slots remain recovery credentials.
Prove the passphrase non-destructively from a trusted running system:

```bash
sudo cryptsetup open --type luks --test-passphrase \
  /dev/disk/by-uuid/<LUKS_UUID>
```

Resolve the real UUID locally; never publish the credential. Success reports
an unlocked keyslot and does not create a mapper because this is a test.

Return to the default normal UKI after the fallback check.

## Rollback

If the reviewed theme fails but the system boots, select the packaged BGRT
theme and rebuild both UKIs:

```bash
sudo plymouth-set-default-theme bgrt
sudo mkinitcpio -P
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

Boot-test normal and fallback again. Only after a successful rollback may the
inactive `/usr/share/plymouth/themes/rogueos` directory be removed or archived.
Do not edit or delete package-owned files under the `bgrt` or `spinner`
directories.

## Completion checklist

- [ ] The repository source and installed descriptor are identical.
- [ ] The custom directory contains only `rogueos.plymouth`.
- [ ] Package-owned Spinner assets remain unmodified.
- [ ] `rogueos` is the selected theme.
- [ ] Normal includes Plymouth, the RogueOS descriptor, and Spinner assets.
- [ ] Fallback contains no Plymouth files and has a textual command line.
- [ ] Both UKIs and executed bootloader files are signed.
- [ ] Normal TPM PIN input, retry behavior, and tuigreet handoff work.
- [ ] Fallback boots successfully without Plymouth.
- [ ] The strong LUKS passphrase has been privately tested.
- [ ] No failed units or relevant journal errors remain.
- [ ] `post-install-26-v1` is created after the documentation commit.

No matching `niri-dotfiles` tag is required: this chapter changes only
system-owned early-boot configuration.

## Sources

- [ArchWiki: Plymouth](https://wiki.archlinux.org/title/Plymouth)
- [Plymouth upstream documentation](https://www.freedesktop.org/wiki/Software/Plymouth/)
- [Arch manual: `crypttab(5)`](https://man.archlinux.org/man/crypttab.5.en)
- [Arch manual: `systemd-cryptenroll(1)`](https://man.archlinux.org/man/systemd-cryptenroll.1.en)
- [Arch manual: `cryptsetup-open(8)`](https://man.archlinux.org/man/cryptsetup-open.8.en)
- [systemd automatic LUKS2 token discovery](https://github.com/systemd/systemd/issues/36293)
