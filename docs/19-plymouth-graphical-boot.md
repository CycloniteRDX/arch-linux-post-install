# 19 — Add Plymouth with a textual fallback UKI

## Goal

Replace the normal boot's changing status text and console LUKS request with a
conservative graphical Plymouth presentation, while making the fallback UKI a
genuinely independent textual recovery path.

The completed design is:

| Boot artifact | Initramfs | Embedded command line | Role |
| --- | --- | --- | --- |
| `/boot/EFI/Linux/arch-linux.efi` | Includes Plymouth | Existing storage policy plus `quiet splash` | Routine graphical boot and graphical LUKS request |
| `/boot/EFI/Linux/arch-linux-fallback.efi` | Skips `autodetect` and Plymouth | Existing storage policy without `quiet splash` | Broad, verbose recovery boot and textual LUKS request |

Plymouth changes presentation only. `sd-encrypt` still opens the LUKS2
container, LVM still activates `vg0`, systemd-boot still selects a signed UKI,
and sbctl still signs each rebuilt EFI executable.

This chapter does not configure TPM2 unlock, hibernation, a custom Plymouth
theme, a UKI `.splash` bitmap, a different initramfs generator, automatic
login, or additional silent-boot parameters. It does not change
`niri-dotfiles`; `post-install-18-v2` remains the latest required user
configuration checkpoint.

The companion handbook guide
[Plymouth, early-boot presentation, and recovery](https://github.com/CycloniteRDX/arch-linux-handbook/blob/main/guides/boot-and-trust/24-plymouth-early-boot-presentation-and-recovery.md)
explains the visual handoffs, renderers, password-agent boundary, alternatives,
and layered diagnosis behind this procedure.

## Status and prerequisites

This chapter is reviewed and awaits hardware validation.

Before applying it:

- chapters 00 through 18 are complete and chapter 18 is hardware-validated;
- the normal and fallback UKIs both boot and pass Secure Boot verification;
- the LUKS passphrase, verified Arch installation USB, and chapter 12 recovery
  material remain available;
- `/boot` is the mounted EFI System Partition, not an ordinary empty directory;
- the systemd-boot menu remains visible for three seconds and its editor is
  disabled;
- the early keyboard layout is the proven US layout;
- AC power is connected and the laptop is initially undocked with its lid open;
- there is enough time to test both UKIs before relying on the machine again.

Run the chapter as `neon` from Kitty. Do not reboot after any failed package,
mkinitcpio, UKI, signing, or inspection command.

## Preserve the ownership and recovery model

| Concern | Owner after this chapter |
| --- | --- |
| EFI selection and three-second menu | systemd-boot |
| UKI assembly | mkinitcpio plus ukify |
| UKI and bootloader signatures | sbctl and the enrolled owner key |
| Early display and password presentation | Plymouth in the normal UKI only |
| LUKS device discovery and unlocking | `sd-encrypt` and systemd-cryptsetup |
| Early keyboard mapping | `keyboard` and `sd-vconsole` |
| LVM and real-root discovery | `lvm2`, `filesystems`, and systemd |
| Graphical login after boot | greetd and tuigreet |

Plymouth must never become a prerequisite for the fallback unlock path. A
broken renderer, theme, or password agent must still leave one signed entry
that exposes ordinary early boot messages and the textual systemd password
request.

The current `editor no` policy remains unchanged. It prevents casual kernel
command-line editing at the menu, so recovery is provided by a separately
built artifact rather than an instruction to append an unsigned option at
boot.

## Audit the known-good boot state

Confirm package, mount, space, service, and boot state before changing a source:

```bash
pacman -Q linux mkinitcpio systemd systemd-ukify sbctl
findmnt -no SOURCE,FSTYPE,OPTIONS /boot
df -h /boot
systemctl --failed --no-pager
sudo sbctl status
sudo sbctl verify
sudo bootctl --esp-path=/boot status
sudo bootctl --esp-path=/boot list
```

`/boot` must be the vfat filesystem on the ESP with its established root-only
masks. Both UKIs and all executed systemd-boot copies must be signed. An
unsigned `/boot/vmlinuz-linux` report remains expected because it is an input
to the signed UKIs, not a boot entry.

Inspect all current build sources and outputs:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/mkinitcpio.d/linux.preset
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Before this chapter:

- both identification commands print `uki`;
- the hook line is
  `base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck`;
- `/etc/kernel/cmdline` contains the same real LUKS UUID after both
  `rd.luks.name=` and `rd.luks.options=`;
- it also contains `discard`, `zswap.enabled=0`,
  `root=/dev/mapper/vg0-root`, and `rw`;
- it contains no `resume=`, `quiet`, or `splash`;
- the preset builds both UKIs and its fallback currently skips only
  `autodetect`;
- both embedded command lines match the current source.

Stop on a placeholder, duplicate parameter, wrong UUID, missing image, failed
unit, unsigned executed artifact, unexpected hook, or unexplained preset
option.

## Create non-overwriting source backups

These three backups are rollback inputs. The fourth file becomes the maintained
fallback command-line source. Confirm none already exists:

```bash
test ! -e /etc/mkinitcpio.conf.before-plymouth
test ! -e /etc/kernel/cmdline.before-plymouth
test ! -e /etc/mkinitcpio.d/linux.preset.before-plymouth
test ! -e /etc/kernel/cmdline-fallback
```

No output and four zero exit statuses are required. If a path exists, stop and
inspect it; do not overwrite evidence from an earlier attempt.

Create the copies:

```bash
sudo cp -a /etc/mkinitcpio.conf /etc/mkinitcpio.conf.before-plymouth
sudo cp -a /etc/kernel/cmdline /etc/kernel/cmdline.before-plymouth
sudo cp -a /etc/mkinitcpio.d/linux.preset /etc/mkinitcpio.d/linux.preset.before-plymouth
sudo cp -a /etc/kernel/cmdline /etc/kernel/cmdline-fallback
```

Verify that the two command-line files are initially identical:

```bash
sudo diff -u /etc/kernel/cmdline.before-plymouth /etc/kernel/cmdline-fallback
```

No output is expected. The fallback source contains hardware identifiers but
not the LUKS passphrase or volume key. It is still machine-specific and must
not be committed to a public repository.

## Install the official Plymouth baseline

Read current Arch news, connect AC, and perform one complete transaction:

```bash
sudo pacman -Syu --needed plymouth
sudo pacdiff --output
```

At the time of review, Plymouth is an official `extra` package and already
provides the mkinitcpio hook, DRM and framebuffer renderers, systemd password
agent integration, installed-system units, and built-in themes. No AUR package,
theme installer, KDE control module, or separate daemon package is needed.

Inspect the installed ownership rather than assuming historical paths:

```bash
pacman -Q plymouth mkinitcpio systemd systemd-ukify sbctl
pacman -Qo /usr/bin/plymouth /usr/bin/plymouthd
pacman -Qo /usr/lib/initcpio/install/plymouth
pacman -Qo /usr/lib/initcpio/hooks/plymouth
pacman -Qo /usr/lib/systemd/system/systemd-ask-password-plymouth.service
mkinitcpio -H plymouth
plymouth-set-default-theme -l
plymouth-set-default-theme
```

If the transaction rebuilt a kernel or UKI, first confirm that the existing
textual images are still signed before editing them again:

```bash
sudo sbctl verify
sudo bootctl --esp-path=/boot list
```

## Add Plymouth to the common hook policy

Open the active mkinitcpio configuration:

```bash
sudo micro /etc/mkinitcpio.conf
```

Replace only the active `HOOKS=` line with:

```bash
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole plymouth block sd-encrypt lvm2 filesystems fsck)
```

The ordering is deliberate:

- `systemd` establishes the systemd-based initramfs;
- `kms`, `keyboard`, and `sd-vconsole` prepare early graphics and input;
- `plymouth` starts before the encrypted device is requested;
- `sd-encrypt` remains the component that opens LUKS;
- `lvm2` and the filesystem hooks retain their storage order.

Do not replace `sd-encrypt` with the BusyBox-oriented `encrypt` hook and do not
add both.

Review the active line:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
```

## Give the two UKIs distinct command lines

Open the normal source:

```bash
sudo micro /etc/kernel/cmdline
```

Append exactly `quiet splash` to the existing single line. Do not reconstruct
the line from the documentation and do not replace either real UUID with a
placeholder. Its shape becomes:

```text
rd.luks.name=<LUKS_UUID>=cryptlvm rd.luks.options=<LUKS_UUID>=discard zswap.enabled=0 root=/dev/mapper/vg0-root rw quiet splash
```

The maintained `/etc/kernel/cmdline-fallback` remains unchanged and therefore
contains the same storage policy without `quiet splash`.

Now open the preset:

```bash
sudo micro /etc/mkinitcpio.d/linux.preset
```

Replace its contents with:

```bash
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

default_uki="/boot/EFI/Linux/arch-linux.efi"
default_cmdline="/etc/kernel/cmdline"

fallback_uki="/boot/EFI/Linux/arch-linux-fallback.efi"
fallback_cmdline="/etc/kernel/cmdline-fallback"
fallback_options="-S autodetect,plymouth"
```

The preset-specific `_cmdline` variables are passed to mkinitcpio's
`--cmdline` option. The fallback `-S` list omits both `autodetect` and the
Plymouth hook: it remains broader in hardware support and independent of the
graphical boot layer.

Review the complete three-source relationship:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
cat /etc/mkinitcpio.d/linux.preset
sudo diff -u /etc/kernel/cmdline.before-plymouth /etc/kernel/cmdline-fallback
```

Required results:

- the common hook list contains Plymouth exactly once before `sd-encrypt`;
- the normal command line preserves every existing storage parameter and adds
  exactly one `quiet` and one `splash`;
- the fallback command line is byte-for-byte equal to the pre-Plymouth source;
- the preset names the two correct UKI outputs;
- only the fallback specifies `-S autodetect,plymouth`;
- neither command line contains `resume=`.

## Select the packaged `bgrt` theme

Use the packaged baseline before considering any custom design:

```bash
sudo plymouth-set-default-theme bgrt
plymouth-set-default-theme
```

The second command must print `bgrt`. This theme can reuse the firmware BGRT
logo and supplies the spinner and password-entry presentation without adding
an external asset source. If BGRT is unavailable or visibly broken during the
hardware test, the packaged `spinner` theme is the first controlled fallback.

Do not use `-R` here. The project deliberately performs the complete
`mkinitcpio -P` operation next so both named UKIs are rebuilt and the sbctl
post-hook output remains visible. If the installed theme helper performs an
intermediate regeneration, let it finish and still run the explicit complete
build below.

## Rebuild and inspect both signed UKIs

Generate both presets:

```bash
sudo mkinitcpio -P
```

Read the full output. The default build must process the Plymouth hook. The
fallback build must report the broader module path and skip both `autodetect`
and `plymouth`. Both UKIs must be created successfully and the sbctl post-hook
must sign each output.

Do not reboot if any build or signing step fails. Inspect the results:

```bash
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
sudo bootctl --esp-path=/boot list
ls -lh /boot/EFI/Linux
```

Confirm all of the following before rebooting:

- both identification commands print `uki`;
- the normal embedded command line ends in `quiet splash`;
- the fallback embeds the pre-Plymouth storage line without either word;
- both preserve the same real LUKS UUID, discard option, zswap policy, root
  mapper, and `rw`;
- both UKIs and every executed systemd-boot copy are signed;
- `arch-linux.efi` remains the default entry;
- `arch-linux-fallback.efi` remains present in the three-second menu.

## Test the normal graphical boot

Save work, leave the laptop undocked with its lid open, and reboot into the
default entry:

```bash
sudo systemctl reboot
```

During this first boot:

1. confirm the systemd-boot menu still appears for three seconds;
2. confirm the BGRT presentation appears at a usable resolution;
3. intentionally enter one wrong LUKS passphrase and require a visible retry;
4. enter the correct passphrase using the established US early keymap;
5. press `Esc` once and confirm detailed boot output can be revealed;
6. allow boot to reach tuigreet, log in, and confirm Plymouth does not remain
   over the greeter or Niri session.

After login, verify the active path:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo cryptsetup status cryptlvm
findmnt /
findmnt /home
findmnt /boot
swapon --show
systemctl --failed --no-pager
systemctl list-units 'plymouth*' --all --no-pager
journalctl -b -u plymouth-start.service -u plymouth-quit.service -u plymouth-quit-wait.service --no-pager
sudo sbctl verify
```

The running command line must contain `quiet splash`; encrypted storage,
filesystems, zram and disk swap must retain their previous state; no Plymouth
unit may be failed.

## Prove the fallback is textual and independent

Save work and reboot. In the systemd-boot menu, select
`arch-linux-fallback.efi` manually.

The fallback test must show ordinary early boot messages and a textual LUKS
request. Use the same passphrase and confirm it reaches tuigreet and Niri.
After login:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
grep -Eo '(^|[[:space:]])(quiet|splash)([[:space:]]|$)' /proc/cmdline || true
sudo cryptsetup status cryptlvm
findmnt /
findmnt /home
systemctl --failed --no-pager
sudo sbctl verify
```

The status must identify the fallback entry and the `grep` command must print
nothing. A graphical prompt, hidden messages, a missing passphrase request, an
unsigned image, or a failure to reach the installed system blocks completion.

Reboot once more and allow the default normal UKI to start. Confirm it is again
the current entry:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo sbctl verify
```

## Complete the hardware presentation matrix

At minimum, validate:

| State | Required observation |
| --- | --- |
| Warm reboot on AC | Plymouth starts, accepts input, and quits cleanly |
| Cold boot on battery | Internal-panel prompt appears and unlock works |
| Lid open and undocked | Canonical 1920×1080 path is legible |
| Wrong passphrase once | Error feedback and retry remain visible |
| `Esc` during normal boot | Detailed messages become available |
| Fallback UKI | Text request works without Plymouth or `quiet splash` |

If a dock and external monitor are available, separately test lid-open docked
and lid-closed docked boots. Early firmware and SimpleDRM routing may expose the
prompt only on the internal panel. Do not declare closed-lid docking supported
until the real hardware proves that at least one observable display presents
the request.

Do not add `plymouth.use-simpledrm=0`, a forced display mode, a delay, or more
quieting parameters merely because one visual handoff flickers. First identify
the failing boundary using the handbook guide.

## Refresh the recovery configuration snapshot

Chapter 19 creates `/etc/kernel/cmdline-fallback` and changes the other three
boot sources. After both UKIs pass their tests, update the encrypted recovery
media using a new snapshot directory. Do not overwrite the pre-Plymouth copy.

The new snapshot must include:

```text
/etc/mkinitcpio.conf
/etc/kernel/cmdline
/etc/kernel/cmdline-fallback
/etc/mkinitcpio.d/linux.preset
/boot/loader/loader.conf
```

It must also retain the existing LUKS header, Secure Boot key, certificate, and
EFI-variable backups from chapter 12. Never place the LUKS header backup,
Secure Boot private keys, or plaintext passphrases on the unencrypted ESP or in
Git.

Unlock and mount the encrypted backup as documented in chapter 12. Confirm the
expected recovery root exists and the new destination does not:

```bash
plymouth_backup_root=/run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
plymouth_config_snapshot="$plymouth_backup_root/system-config-post-install-19"
test -d "$plymouth_backup_root"
test ! -e "$plymouth_config_snapshot"
```

Stop if either test fails. Then create the new root-only snapshot:

```bash
sudo install -d -m 0700 "$plymouth_config_snapshot"
sudo cp --archive \
  /etc/mkinitcpio.conf \
  /etc/kernel/cmdline \
  /etc/kernel/cmdline-fallback \
  /etc/mkinitcpio.d/linux.preset \
  /boot/loader/loader.conf \
  "$plymouth_config_snapshot/"
sudo find "$plymouth_config_snapshot" -maxdepth 1 -type f -printf '%M %u:%g %f\n'
```

This adds a current configuration snapshot; it does not replace the original
recovery directory or the independently tested Restic data backup.
Review the copy, synchronize pending writes, and close the encrypted backup by
following chapter 12's established unmount sequence.

## Diagnosis and safe recovery

### Normal boot becomes black before the password prompt

Press `Esc`, open the laptop lid, and disconnect any dock. Do not type the LUKS
passphrase blindly into a display state you cannot identify. Restart and select
the textual fallback entry.

If fallback works, the shared storage and kernel path are probably healthy;
inspect the normal command line, Plymouth units, early DRM messages, and the
saved source files before changing anything.

### Plymouth shows but no password request appears

Use `Esc` and then the fallback entry. Do not remove `sd-encrypt`, change the
LUKS passphrase, enroll TPM2, or add a second encryption hook. Confirm the
Plymouth hook precedes `sd-encrypt` and that the package supplies the systemd
password agent.

### The normal UKI fails but fallback boots

From the fallback session, first confirm `/boot` and the signatures:

```bash
findmnt /boot
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo sbctl verify
```

Restore the three known-good sources:

```bash
sudo cp -a /etc/mkinitcpio.conf.before-plymouth /etc/mkinitcpio.conf
sudo cp -a /etc/kernel/cmdline.before-plymouth /etc/kernel/cmdline
sudo cp -a /etc/mkinitcpio.d/linux.preset.before-plymouth /etc/mkinitcpio.d/linux.preset
```

Review them, rebuild both textual UKIs, and verify every output:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/mkinitcpio.d/linux.preset
sudo mkinitcpio -P
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
sudo bootctl --esp-path=/boot list
```

The restored preset no longer refers to `/etc/kernel/cmdline-fallback`; leave
that now-unused file in place until the rollback has been boot-tested and
recorded. The Plymouth package may also remain installed while its hook is
absent. Package removal is a later cleanup step, not part of emergency repair.

### Neither UKI boots

Use the verified Arch installation USB and chapter 12's mount-and-chroot
procedure. Restore or correct the saved sources inside the installed system,
run `mkinitcpio -P`, inspect both UKIs, and verify sbctl signatures before
leaving the chroot. Do not recreate LUKS, clear Secure Boot keys, reformat the
ESP, or reinstall Arch for a presentation-layer failure.

## Completion checklist

- [ ] Chapters 00 through 18 remain healthy and `post-install-18-v2` remains the dotfiles checkpoint.
- [ ] The official Plymouth package and its mkinitcpio/password-agent files are installed.
- [ ] The three pre-Plymouth sources are preserved without overwriting an earlier attempt.
- [ ] `/etc/kernel/cmdline-fallback` preserves the exact pre-Plymouth storage policy.
- [ ] The common hook list contains Plymouth once before `sd-encrypt`.
- [ ] The normal source adds only `quiet splash` to the established command line.
- [ ] The fallback preset skips both `autodetect` and Plymouth.
- [ ] Both UKIs rebuild successfully and are signed by the existing sbctl workflow.
- [ ] The normal UKI embeds `quiet splash`; the fallback embeds neither parameter.
- [ ] The BGRT theme displays a usable LUKS request on the internal panel.
- [ ] One wrong passphrase produces visible feedback and a working retry.
- [ ] `Esc` reveals detailed output during the normal path.
- [ ] Normal boot reaches tuigreet and Niri without a stuck splash or failed unit.
- [ ] The textual fallback unlocks LUKS and reaches the installed system independently.
- [ ] A warm reboot and cold battery boot both pass.
- [ ] Dock and external-monitor behavior are recorded if that hardware is available.
- [ ] A new encrypted recovery snapshot includes `cmdline-fallback` and all current boot sources.
- [ ] TPM2 enrollment, custom themes, UKI bitmap, hibernation, and extra silent options remain absent.

## Sources

- [Arch package: Plymouth](https://archlinux.org/packages/extra/x86_64/plymouth/)
- [Arch package file list: Plymouth](https://archlinux.org/packages/extra/x86_64/plymouth/files/)
- [ArchWiki: Plymouth](https://wiki.archlinux.org/title/Plymouth)
- [`plymouth(8)`](https://man.archlinux.org/man/plymouth.8.en)
- [`plymouthd(8)`](https://man.archlinux.org/man/plymouthd.8.en)
- [`plymouth-set-default-theme(1)`](https://man.archlinux.org/man/plymouth-set-default-theme.1.en)
- [`mkinitcpio(8)`](https://man.archlinux.org/man/mkinitcpio.8.en)
- [`bootctl(1)`](https://man.archlinux.org/man/bootctl.1.en)
- [`systemd-ask-password(1)`](https://man.archlinux.org/man/systemd-ask-password.1.en)
- [sbctl upstream repository](https://github.com/Foxboron/sbctl)

## Next step

Hardware-validate this chapter before changing the boot trust or unlock model.
Once normal and textual fallback boot are both stable, the next planned
extension is TPM2-bound LUKS unlocking with the strong passphrase retained as
the recovery credential.
