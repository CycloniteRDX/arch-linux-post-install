# 20 — Add TPM2-bound LUKS unlock with a PIN

## Goal

Add a convenient normal-boot unlock route whose random LUKS2 credential is
sealed to this ThinkPad's TPM2, the current Secure Boot policy, and an
update-aware signed UKI policy. Require a unique TPM PIN so possession of the
laptop alone does not unlock the root filesystem.

The completed credential model is:

| Route | Required evidence or credential | Intended use |
| --- | --- | --- |
| Normal UKI TPM path | This TPM2, raw PCR 7, a valid signed PCR 11 policy, and the TPM PIN | Routine boot |
| Normal UKI manual fallback | Existing strong LUKS passphrase | TPM or policy refusal |
| Textual fallback UKI | Existing strong LUKS passphrase or generated recovery key | Independent boot recovery |
| Arch installation ISO | Existing strong LUKS passphrase or generated recovery key | Repair when neither UKI reaches a usable prompt |

TPM enrollment adds one LUKS keyslot and one `systemd-tpm2` token. It does not
replace the current passphrase, reformat LUKS, clear the TPM, copy the
passphrase into the TPM, or make the fallback UKI depend on TPM support.
It changes no user configuration; `post-install-18-v2` remains the latest
required `niri-dotfiles` checkpoint.

The companion handbook guide
[TPM2-bound LUKS unlock, measured-boot policy, and recovery](https://github.com/CycloniteRDX/arch-linux-handbook/blob/main/guides/boot-and-trust/25-tpm2-bound-luks-unlock-measured-boot-and-recovery.md)
explains PCRs, tokens and keyslots, signed policy, the threat model, update
behavior, alternatives, and recovery boundaries behind this procedure.

## Status and safety gate

This chapter passed complete hardware validation on the target ThinkPad T14
Gen 1 AMD on 2026-09-06. The validated result includes the normal PIN path,
the independent textual fallback, the signed-PCR update test without TPM
reenrollment, and the manual recovery paths. Validation also confirmed that
the TPM identification probe needs elevated device access on this ThinkPad and
exposed the two version-specific command corrections documented below.

Before applying it:

- chapters 00 through 19 are complete and chapter 19 is hardware-validated;
- the normal Plymouth UKI and textual fallback UKI both cold-boot correctly;
- the fallback accepts the strong LUKS passphrase without Plymouth;
- Secure Boot is enabled and every executed boot artifact verifies;
- the verified Arch installation USB and chapter 12 chroot procedure remain
  available;
- the encrypted external recovery disk is available and has enough space for
  several dated LUKS2 header backups;
- the existing LUKS passphrase is known and has been tested;
- AC power is connected for package and UKI work;
- there is enough time to complete every checkpoint and controlled reboot.

Do not run this chapter as one pasted block. It intentionally stops for three
boot tests before normal TPM unlock is accepted:

1. rebuild UKIs with signed PCR policy but no TPM request or token;
2. add the TPM request to the normal UKI but still keep no token;
3. enroll one token only after both earlier states remain manually bootable.

Stop after any unexpected device, UUID, keyslot, token, PCR, package, build,
signature, prompt, or boot result. A passphrase prompt is a safe fallback; it
is not a reason to weaken the policy.

Run the chapter as `neon` from Kitty unless a step explicitly enters the boot
menu or recovery environment. Never paste a passphrase, recovery key, TPM PIN,
private key, LUKS header, or complete token dump into Git, chat, screenshots,
shell arguments, or synchronized notes.

## Preserve the ownership model

| Concern | Owner after this chapter |
| --- | --- |
| EFI selection and menu | systemd-boot |
| UKI and initramfs assembly | mkinitcpio plus ukify |
| Whole-UKI Secure Boot signatures | sbctl |
| Expected PCR 11 signatures | ukify plus a dedicated per-machine PCR-policy key |
| Normal graphical prompt | Plymouth password agent |
| LUKS keyslots and JSON token | cryptsetup plus systemd-cryptenroll |
| TPM policy evaluation | systemd-cryptsetup plus this ThinkPad's TPM2 |
| Manual recovery | LUKS passphrase, recovery key, textual UKI, and Arch ISO |

The Secure Boot private key and the PCR-policy private key have different
jobs. sbctl continues to sign the complete EFI executable. ukify uses a new,
separate key to sign expected PCR 11 values before sbctl signs the completed
UKI. Do not put sbctl's private key into `/etc/kernel/uki.conf` and do not reuse
the PCR-policy key between ThinkPads.

## Audit the known-good pre-TPM state

Confirm the installed packages, ESP, boot menu, signatures, storage, and
services:

```bash
pacman -Q linux mkinitcpio systemd systemd-ukify cryptsetup sbctl plymouth
findmnt -no SOURCE,FSTYPE,OPTIONS /boot
df -h /boot
systemctl --failed --no-pager
sudo sbctl status
sudo sbctl verify
sudo bootctl --esp-path=/boot status
sudo bootctl --esp-path=/boot list
sudo cryptsetup status cryptlvm
findmnt /
findmnt /home
swapon --show
```

Required foundations are the established vfat ESP at `/boot`, Secure Boot in
User Mode, signed normal and fallback UKIs, active `cryptlvm`, mounted `vg0`
root and home, and the existing zram plus encrypted disk-swap policy. Stop on
an unsigned executed image, failed unit, unexpected mount, or storage change.

Inspect the exact UKI sources and outputs:

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
cat /etc/mkinitcpio.d/linux.preset
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-identify /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Before this chapter:

- the common hooks include `systemd`, Plymouth, and `sd-encrypt` in the
  chapter 19 order;
- normal embeds the real LUKS UUID, `discard`, `zswap.enabled=0`, the `vg0`
  root mapper, `rw quiet splash`, and no TPM option;
- fallback embeds the same storage identifiers but omits `quiet`, `splash`,
  and every TPM option;
- the preset skips `autodetect,plymouth` only for fallback;
- both identification commands print `uki`.

Resolve the real UUID independently and compare it with both sources:

```bash
luks_uuid="$(sudo cryptsetup luksUUID /dev/nvme0n1p2)"
printf 'LUKS UUID: %s\n' "$luks_uuid"
grep -F "rd.luks.name=$luks_uuid=cryptlvm" /etc/kernel/cmdline
grep -F "rd.luks.options=$luks_uuid=discard" /etc/kernel/cmdline
grep -F "rd.luks.name=$luks_uuid=cryptlvm" /etc/kernel/cmdline-fallback
grep -F "rd.luks.options=$luks_uuid=discard" /etc/kernel/cmdline-fallback
```

All four `grep` commands must print one match. The shell variable is only a
temporary inspection value. Never type `<LUKS_UUID>` literally or copy a UUID
from another machine.

## Audit TPM2 discovery and current credentials

Use systemd's read-only TPM probes:

```bash
systemd-analyze has-tpm2
sudo systemd-analyze identify-tpm2
systemd-cryptenroll --tpm2-device=list
systemd-analyze pcrs 7 11
ls -l /dev/tpm0 /dev/tpmrm0
journalctl -b -k --grep='tpm|ima' --no-pager
```

Continue only when the firmware and kernel expose one intended TPM2 and
systemd can identify it. `identify-tpm2` reads information from the TPM device;
on this installed profile it requires `sudo`, even though `has-tpm2` can run as
the regular user. PCR 7 and PCR 11 must be readable in a SHA-256 bank. Review
journal output locally because it can reveal platform identifiers.

Do not enable, disable, initialize, reset, provision, or clear the TPM from
firmware or Linux during this audit. Do not install `tpm2-abrmd`; this design
uses the kernel resource-manager device.

Inventory LUKS2 slots and tokens:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup luksDump /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
```

Enter the current strong LUKS passphrase for the last command. It must succeed
without creating another mapping. Record slot *types* privately, not their
full metadata.

At least one known password slot must exist. Stop if a TPM2 token already
exists and its origin is not fully understood. Do not wipe an unexplained
token merely to make the later command look tidy.

## Install TPM2 userspace support through a complete upgrade

Review Arch news as in chapter 02, then install the official TPM2 software
stack with a complete transaction:

```bash
sudo pacman -Syu tpm2-tss
```

Read the entire transaction and any `.pacnew` or UKI-generation output. Do not
add `tpm2-tools` merely for enrollment; it remains an optional diagnostic
package. Confirm the matching components:

```bash
pacman -Q systemd systemd-ukify cryptsetup tpm2-tss mkinitcpio
systemd-analyze has-tpm2
systemd-cryptenroll --tpm2-device=list
sudo sbctl verify
```

If the transaction updated the kernel, systemd, mkinitcpio, ukify, cryptsetup,
Plymouth, or a boot input, inspect both UKIs and cold boot the normal and
fallback entries once before continuing. The chapter does not layer TPM work
on top of an untested boot-critical upgrade.

## Create the before-TPM recovery checkpoint

Identify the encrypted external disk again instead of relying on a remembered
`/dev/sdX` name:

```bash
ls -l /dev/disk/by-id/usb-*
backup_disk=/dev/disk/by-id/usb-REPLACE_WITH_THE_EXACT_WHOLE_DISK_ID
backup_partition="${backup_disk}-part1"
test -b "$backup_disk"
test -b "$backup_partition"
readlink -f "$backup_disk"
lsblk -d -o NAME,PATH,SIZE,TYPE,MODEL,SERIAL,TRAN,RM \
  "$(readlink -f "$backup_disk")"
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS \
  "$backup_partition"
```

Replace the deliberately invalid assignment with the exact persistent entry
for the whole USB disk, not one ending in `-part1`. The resolved whole device
may currently be `/dev/sda`, but it must never be selected from that temporary
name alone. Its model, serial, capacity, `TYPE=disk`, and `TRAN=usb` must match
the physical backup disk; the partition must be LUKS2 labelled
`ARCH-BACKUP-LUKS`.

Unlock the LUKS partition, discover its cleartext mapping, and mount the ext4
filesystem if the desktop did not mount it automatically:

```bash
udisksctl unlock -b "$backup_partition"
backup_mapping="$(
  lsblk -nrpo PATH,TYPE "$backup_partition" |
  awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET || true)"
if [ -z "$backup_mount" ]; then
  udisksctl mount -b "$backup_mapping"
  backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
fi
test "$backup_mount" = /run/media/neon/ARCH-BACKUP
```

`udisksctl unlock` exposes the decrypted block device; it does not guarantee
that the filesystem has also been mounted. The mapping may be named
`/dev/mapper/ARCH-BACKUP-LUKS`, `/dev/mapper/luks-...`, or `/dev/dm-*`, so the
commands discover it rather than assuming a name.

Verify the real mount rather than trusting the directory name:

```bash
findmnt --target "$backup_mount"
df -hT "$backup_mount"
test "$(findmnt -nr -T "$backup_mount" -o TARGET)" = "$backup_mount"
test "$(findmnt -nr -T "$backup_mount" -o FSTYPE)" = ext4
findmnt -nr -T "$backup_mount" -o OPTIONS | tr ',' '\n' | grep -Fx rw
```

The source must be the external encrypted mapping and the filesystem must be
ext4 mounted read-write. Define a new, non-overwriting checkpoint:

```bash
tpm_backup_root="$backup_mount/rogue-thinkpad-recovery"
tpm_before="$tpm_backup_root/post-install-20-before-tpm2"
test -d "$tpm_backup_root"
test ! -e "$tpm_before"
```

Both tests must succeed. If the destination already exists, stop and determine
whether this chapter already ran. A later deliberate repeat must use a new,
consistently named checkpoint throughout; never overwrite an older header.

Create the root-only directory and capture the current header plus boot
configuration:

```bash
sudo install -d -m 0700 "$tpm_before"
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 \
  --header-backup-file "$tpm_before/nvme0n1p2-luks2-header-before-tpm.img"
sudo cp --archive \
  /etc/mkinitcpio.conf \
  /etc/kernel/cmdline \
  /etc/kernel/cmdline-fallback \
  /etc/mkinitcpio.d/linux.preset \
  /boot/loader/loader.conf \
  "$tpm_before/"
sudo sh -c 'cd "$1" && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS' sh "$tpm_before"
sudo chmod -R go-rwx "$tpm_before"
sudo chown -R root:root "$tpm_before"
sudo sh -c 'cd "$1" && sha256sum --check SHA256SUMS' sh "$tpm_before"
```

Inspect the backup as a file and compare UUIDs without restoring it:

```bash
sudo cryptsetup luksDump "$tpm_before/nvme0n1p2-luks2-header-before-tpm.img"
sudo cryptsetup luksUUID /dev/nvme0n1p2
sudo cryptsetup luksUUID "$tpm_before/nvme0n1p2-luks2-header-before-tpm.img"
```

The UUIDs must match. `luksHeaderRestore` is a destructive recovery operation,
not a validation command.

## Add or prove an independent generated recovery key

Run the concise inventory again:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
```

If a known, securely recorded recovery-key slot already exists, do not create
another. Test that key with:

```bash
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
```

If no recovery-key slot exists, prepare a paper or other offline secret store
away from the laptop, then enroll one:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2 --recovery-key
```

Authorize the change with the strong LUKS passphrase. Record the generated key
offline immediately, check every character, and keep it separate from the
encrypted backup disk. Do not photograph the QR code into a synchronized
library, copy the key to the clipboard, save terminal output, or leave it in a
plaintext file.

Test the new recovery key before any TPM enrollment:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
```

Enter the generated recovery key for the second command. Then test the strong
passphrase again with the same command. Both must work independently.

Capture the new credential state without replacing the earlier header:

```bash
tpm_backup_root=/run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
tpm_before="$tpm_backup_root/post-install-20-before-tpm2"
findmnt --target /run/media/neon/ARCH-BACKUP
test "$(findmnt -nr -T /run/media/neon/ARCH-BACKUP -o TARGET)" = /run/media/neon/ARCH-BACKUP
test -d "$tpm_before"
test ! -e "$tpm_before/nvme0n1p2-luks2-header-after-recovery-key.img"
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 \
  --header-backup-file "$tpm_before/nvme0n1p2-luks2-header-after-recovery-key.img"
sudo sh -c 'cd "$1" && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS' sh "$tpm_before"
sudo sh -c 'cd "$1" && sha256sum --check SHA256SUMS' sh "$tpm_before"
```

Do not continue until the password and recovery routes both pass.

## Create a dedicated PCR-policy key

Confirm that this project has not already created any of its target files:

```bash
test ! -e /etc/kernel/uki.conf
test ! -e /etc/systemd/tpm2-pcr-private-key-initrd.pem
test ! -e /etc/systemd/tpm2-pcr-public-key-initrd.pem
```

No output and three zero exit statuses are required. If any path exists, stop
and inspect its ownership, origin, references, and backup before deciding
whether this chapter already ran. Do not overwrite or regenerate a private key
that may already authorize enrolled tokens.

Create the ukify configuration:

```bash
sudo micro /etc/kernel/uki.conf
```

Enter exactly:

```ini
[UKI]
PCRBanks=sha256

[PCRSignature:initrd]
Phases=enter-initrd enter-initrd:leave-initrd enter-initrd:leave-initrd:sysinit enter-initrd:leave-initrd:sysinit:ready
PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key-initrd.pem
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key-initrd.pem
```

The four signed phase paths use one initrd-policy key and cover the useful PCR
11 states from early initrd through the fully started system. The first value,
`enter-initrd`, is the one available when the encrypted root is unlocked. The
later values let `systemd-cryptenroll` validate the same signed policy safely
when it is invoked after login and PCR 11 has already reached `ready`.

This file declares PCR signing only. It deliberately contains no
`SecureBootPrivateKey`, `SecureBootCertificate`, `SecureBootSigningTool`, or
sbctl path.

Review it, generate the new pair once, and enforce its modes:

```bash
sudo cat /etc/kernel/uki.conf
sudo ukify genkey --config=/etc/kernel/uki.conf
sudo chown root:root \
  /etc/kernel/uki.conf \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem
sudo chmod 0644 /etc/kernel/uki.conf
sudo chmod 0600 /etc/systemd/tpm2-pcr-private-key-initrd.pem
sudo chmod 0644 /etc/systemd/tpm2-pcr-public-key-initrd.pem
sudo stat -c '%A %U:%G %s %n' \
  /etc/kernel/uki.conf \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem
```

The private key must be `-rw------- root:root` and non-empty. It authorizes
future UKI measurements under the enrolled TPM policy and is therefore a
secret even though it cannot decrypt the disk by itself.

Back up the policy material to the already verified encrypted checkpoint:

```bash
tpm_backup_root=/run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
tpm_before="$tpm_backup_root/post-install-20-before-tpm2"
findmnt --target /run/media/neon/ARCH-BACKUP
test "$(findmnt -nr -T /run/media/neon/ARCH-BACKUP -o TARGET)" = /run/media/neon/ARCH-BACKUP
test -d "$tpm_before"
test ! -e "$tpm_before/pcr-policy"
sudo install -d -m 0700 "$tpm_before/pcr-policy"
sudo cp --archive \
  /etc/kernel/uki.conf \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem \
  "$tpm_before/pcr-policy/"
sudo sh -c 'cd "$1" && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS' sh "$tpm_before"
sudo chmod -R go-rwx "$tpm_before"
sudo chown -R root:root "$tpm_before"
sudo sh -c 'cd "$1" && sha256sum --check SHA256SUMS' sh "$tpm_before"
```

Each ThinkPad must generate and protect its own key pair. Git contains only
the paths and policy, never either machine's generated files.

Do not replace this with ukify's two-key example merely because that example
exists. Two PCR signing keys compartmentalize distinct secrets consumed in
the initrd and later system phases. This design protects only the root LUKS
credential during initrd, so it has no separate late-boot secret that would
benefit from a second key. The additional signatures do not bypass PCR 7,
Secure Boot, or the TPM PIN, and only the `enter-initrd` state can match when
root unlock is requested.

Keep this configuration after enrollment. Reducing `Phases=` back to only
`enter-initrd` would not strengthen the actual early root unlock, but a later
re-enrollment performed from the fully booted system would again lack a signed
policy for its current PCR 11 state. Deleting the file would also stop future
UKI regenerations from receiving updated PCR signatures.

Synchronize pending writes and close the complete external stack in reverse
order before the controlled boot tests:

```bash
sync
udisksctl unmount -b "$backup_mapping"
udisksctl lock -b "$backup_partition"
udisksctl power-off -b "$(readlink -f "$backup_disk")"
```

The unmount and lock operations must succeed before `power-off` or physical
disconnection. `unmount` targets the cleartext ext4 mapping, `lock` targets the
encrypted partition, and `power-off` targets the whole physical disk. For the
current temporary names that usually means mapping -> `/dev/sda1` -> `/dev/sda`;
`udisksctl power-off -b /dev/sda1` would be the wrong layer. If an operation
reports that the device is busy, find the process with:

```bash
sudo lsof +f -- "$backup_mount"
```

Close that process and retry without `--force`. Disconnect the disk only after
`udisksctl power-off` reports success.

## Checkpoint 1 — Build signed-PCR UKIs without requesting TPM unlock

The existing mkinitcpio UKI path automatically discovers
`/etc/kernel/uki.conf`. Build both presets:

```bash
sudo mkinitcpio -P
```

Read the full output. ukify must add PCR policy sections and the established
sbctl post hook must still sign both completed images. Do not reboot after a
build or signing error.

Inspect both artifacts:

```bash
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
sudo bootctl --esp-path=/boot list
```

Required results:

- both UKIs contain non-empty `.pcrsig` and `.pcrpkey` sections;
- the signatures describe SHA-256 PCR 11 and include `enter-initrd`,
  `enter-initrd:leave-initrd`, `enter-initrd:leave-initrd:sysinit`, and
  `enter-initrd:leave-initrd:sysinit:ready`;
- both contain the same new PCR-policy public key;
- normal still embeds `discard` without `tpm2-device=auto` and retains
  `quiet splash`;
- fallback retains `discard` without TPM, `quiet`, or `splash`;
- both UKIs and the active systemd-boot copies verify under sbctl.

Cold boot the normal UKI. It must still request and accept the full LUKS
passphrase because no TPM token or request exists yet. After login, prove that
systemd-stub handed the signed policy into the running system:

```bash
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
sudo sha256sum \
  /run/systemd/tpm2-pcr-public-key.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem
sudo sbctl verify
systemctl --failed --no-pager
```

The reboot before this block is mandatory. Files under `/run/systemd/` were
exported by the UKI used for the current boot; running `mkinitcpio -P` or
editing `/etc/kernel/uki.conf` cannot replace them inside an already running
system. Comparing the hashes before booting the rebuilt normal UKI would
compare stale runtime material and could produce a misleading mismatch.

The two public-key hashes must match. Stop if a runtime file is missing or the
hashes differ. A UKI section visible on disk is not sufficient by itself.

Next, boot the textual fallback once. It must still display ordinary early
messages and accept the full LUKS passphrase. Return to the normal entry before
continuing.

## Checkpoint 2 — Request TPM only from the normal UKI

Create one non-overwriting rollback copy of the normal command line:

```bash
test ! -e /etc/kernel/cmdline.before-tpm2
sudo cp -a /etc/kernel/cmdline /etc/kernel/cmdline.before-tpm2
```

Stop if the first command reports that a file already exists. Open the active
normal source:

```bash
sudo micro /etc/kernel/cmdline
```

Change only its existing per-device option from:

```text
rd.luks.options=<LUKS_UUID>=discard
```

to:

```text
rd.luks.options=<LUKS_UUID>=discard,tpm2-device=auto
```

Keep the real UUID and every other reviewed parameter. The complete shape is:

```text
rd.luks.name=<LUKS_UUID>=cryptlvm rd.luks.options=<LUKS_UUID>=discard,tpm2-device=auto zswap.enabled=0 root=/dev/mapper/vg0-root rw quiet splash
```

`<LUKS_UUID>` remains documentation syntax and must never be pasted literally.
The comma is significant: `discard` and `tpm2-device=auto` are options for one
device, not two duplicate `rd.luks.options=` parameters.

Do not edit `/etc/kernel/cmdline-fallback`. Resolve the UUID again in this new
shell if an earlier checkpoint rebooted the machine, then review the
separation:

```bash
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
luks_uuid="$(sudo cryptsetup luksUUID /dev/nvme0n1p2)"
grep -F "rd.luks.options=$luks_uuid=discard,tpm2-device=auto" /etc/kernel/cmdline
grep -F "rd.luks.options=$luks_uuid=discard" /etc/kernel/cmdline-fallback
! grep -F 'tpm2-device=' /etc/kernel/cmdline-fallback
```

The first two `grep` checks must print their intended single lines. The last
command must print nothing.

Rebuild and inspect both UKIs again:

```bash
sudo mkinitcpio -P
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

Normal must now embed the combined option; fallback must not contain
`tpm2-device=auto`. Both must retain valid PCR policy and Secure Boot
signatures.

Cold boot the normal UKI before enrolling any token. It should still fall back
to the full LUKS passphrase. After login:

```bash
cat /proc/cmdline
luks_uuid="$(sudo cryptsetup luksUUID /dev/nvme0n1p2)"
grep -F "rd.luks.options=$luks_uuid=discard,tpm2-device=auto" /proc/cmdline
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo sbctl verify
systemctl --failed --no-pager
```

The active command line must contain the TPM request, but the inventory must
still show no TPM2 slot. This proves the normal-only option does not remove the
manual path.

Boot the fallback again and verify:

```bash
cat /proc/cmdline
grep -Eo 'tpm2-device=[^[:space:]]+' /proc/cmdline || true
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo sbctl verify
```

The fallback must remain textual, accept the strong passphrase, and print no
TPM option. Return to the normal entry with the full passphrase before the
enrollment stage.

## Choose the TPM PIN

Prepare a unique PIN that is used only for this TPM-bound root unlock. Despite
its name, systemd permits non-numeric characters.

The PIN must not equal the LUKS passphrase, recovery key, login password, sudo
password, keyring password, or another device PIN. Do not write it beside the
laptop, store it in Git or dotfiles, or place it in a shell variable or command
argument.

Incorrect attempts increment the TPM's global dictionary-attack protection
and can delay TPM-backed operations. This chapter never asks for an intentional
wrong-PIN test. Error recovery is tested through the independent fallback UKI
and LUKS credentials.

## Enroll one TPM2 token last

Confirm that the current boot is the normal entry and that every prerequisite
still holds:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
sudo sha256sum \
  /run/systemd/tpm2-pcr-public-key.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem
systemd-analyze has-tpm2
systemd-analyze pcrs 7 11
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
sudo sbctl verify
```

Use the strong passphrase for the test. The runtime and local public-key hashes
must match, the normal command line must request TPM2, and no unexplained TPM2
slot may exist.

Enroll the token with every policy input explicit:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2 \
  --wipe-slot=tpm2 \
  --tpm2-device=auto \
  --tpm2-pcrs=7:sha256 \
  --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key-initrd.pem \
  --tpm2-public-key-pcrs=11 \
  --tpm2-signature=/run/systemd/tpm2-pcr-signature.json \
  --tpm2-with-pin=yes
```

First enter the existing strong LUKS passphrase to authorize the header
change, then enter and confirm the new unique TPM PIN. Stop on any warning
about the device, PCR signature, public key, unsupported algorithm, PIN, or
slot operation.

The asymmetry is intentional. `--tpm2-pcrs=7:sha256` selects one raw PCR and
its bank. On the systemd version validated on this ThinkPad,
`--tpm2-public-key-pcrs=` expects the signed PCR mask and rejects
`11:sha256` with `Not expecting hash algorithm specification in PCR mask
value`. Use `11`; `PCRBanks=sha256` and the generated `.pcrsig` policy
still make the signed PCR 11 authorization SHA-256. Omitting the bank from this
mask does not downgrade it to SHA-1.

The command creates the replacement TPM2 enrollment before wiping only older
TPM2 slots. It must never be changed to `--wipe-slot=password`,
`--wipe-slot=recovery`, `--wipe-slot=all`, or a guessed numeric slot.

Inspect the result without rebooting:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup luksDump /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
sudo sbctl verify
```

Required credential types are at least:

- one tested password slot;
- one tested recovery-key slot;
- exactly one intended TPM2 slot and `systemd-tpm2` token;
- TPM policy metadata describing PIN use, raw SHA-256 PCR 7, and signed
  SHA-256 PCR 11.

Use the strong passphrase for the test command. Do not proceed if it stopped
working.

Create a distinct after-enrollment recovery checkpoint:

The preceding reboots discarded the shell variables. Repeat the identification,
unlock, mapping discovery, and conditional mount sequence here so this
checkpoint does not depend on another chapter:

```bash
ls -l /dev/disk/by-id/usb-*
backup_disk=/dev/disk/by-id/usb-REPLACE_WITH_THE_EXACT_WHOLE_DISK_ID
backup_partition="${backup_disk}-part1"
test -b "$backup_disk"
test -b "$backup_partition"
readlink -f "$backup_disk"
lsblk -d -o NAME,PATH,SIZE,TYPE,MODEL,SERIAL,TRAN,RM \
  "$(readlink -f "$backup_disk")"
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS \
  "$backup_partition"
udisksctl unlock -b "$backup_partition"
backup_mapping="$(
  lsblk -nrpo PATH,TYPE "$backup_partition" |
  awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET || true)"
if [ -z "$backup_mount" ]; then
  udisksctl mount -b "$backup_mapping"
  backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
fi
test "$backup_mount" = /run/media/neon/ARCH-BACKUP
findmnt --target "$backup_mount"
df -hT "$backup_mount"
test "$(findmnt -nr -T "$backup_mount" -o FSTYPE)" = ext4
findmnt -nr -T "$backup_mount" -o OPTIONS | tr ',' '\n' | grep -Fx rw
tpm_backup_root="$backup_mount/rogue-thinkpad-recovery"
tpm_after="$tpm_backup_root/post-install-20-after-tpm2"
test -d "$tpm_backup_root"
test ! -e "$tpm_after"
sudo install -d -m 0700 "$tpm_after/pcr-policy"
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 \
  --header-backup-file "$tpm_after/nvme0n1p2-luks2-header-after-tpm.img"
sudo cp --archive \
  /etc/mkinitcpio.conf \
  /etc/kernel/cmdline \
  /etc/kernel/cmdline-fallback \
  /etc/kernel/cmdline.before-tpm2 \
  /etc/kernel/uki.conf \
  /etc/mkinitcpio.d/linux.preset \
  /boot/loader/loader.conf \
  "$tpm_after/"
sudo cp --archive \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem \
  "$tpm_after/pcr-policy/"
sudo sh -c 'cd "$1" && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS' sh "$tpm_after"
sudo chmod -R go-rwx "$tpm_after"
sudo chown -R root:root "$tpm_after"
sudo sh -c 'cd "$1" && sha256sum --check SHA256SUMS' sh "$tpm_after"
```

Do not copy the actual PIN or recovery key into this directory. Close and power
off the complete stack before the boot tests:

```bash
sync
udisksctl unmount -b "$backup_mapping"
udisksctl lock -b "$backup_partition"
udisksctl power-off -b "$(readlink -f "$backup_disk")"
```

All three operations must succeed. Do not unplug the drive after merely
unmounting ext4: locking removes the cleartext mapping, and powering off the
whole disk flushes and deconfigures the USB device.

## Test the normal TPM2 plus PIN path

Save work, disconnect removable storage, leave the lid open, and cold boot the
default normal UKI with Secure Boot enabled.

Verify the sequence:

1. systemd-boot selects `arch-linux.efi`;
2. Plymouth presents a TPM2 token PIN request, not a request to disclose the
   full LUKS passphrase;
3. the unique correct PIN opens `cryptlvm`;
4. the system reaches tuigreet and the normal Niri session;
5. root, home, zram, disk swap, networking, and the desktop remain healthy.

Do not intentionally enter a wrong PIN. If the PIN route refuses the current
state, enter the strong LUKS passphrase when the password agent falls back, or
restart into the textual fallback. A working passphrase after TPM refusal is
the intended safety behavior.

After a successful PIN boot:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo cryptsetup status cryptlvm
sudo systemd-cryptenroll /dev/nvme0n1p2
systemd-analyze pcrs 7 11
findmnt /
findmnt /home
findmnt /boot
swapon --show
sudo sbctl status
sudo sbctl verify
systemctl --failed --no-pager
journalctl -b -u 'systemd-cryptsetup@cryptlvm.service' --no-pager
```

Review the journal locally before sharing it. UUIDs, slot numbers, policy
hashes, and platform details are not secret keys, but they describe the
machine's security design.

## Prove the independent textual fallback

Reboot and manually select `arch-linux-fallback.efi`. It must:

- show the textual early-boot path without Plymouth;
- request a full LUKS credential rather than the TPM PIN;
- accept the existing strong passphrase;
- omit `tpm2-device=auto`, `quiet`, and `splash` from the running command line;
- reach the installed system with the broader initramfs;
- leave the TPM token present but unused by this boot path.

After login:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
grep -Eo '(^|[[:space:]])(quiet|splash)([[:space:]]|$)|tpm2-device=[^[:space:]]+' /proc/cmdline || true
sudo cryptsetup status cryptlvm
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
sudo sbctl verify
systemctl --failed --no-pager
```

The `grep` command must print nothing. Use the strong passphrase for the test.
Schedule a separate private fallback boot using the generated recovery key;
do not perform it while screen recording or in public.

Finally reboot the default normal UKI and confirm the TPM PIN route works a
second time. This proves both paths remain deliberate rather than one being an
accidental first-boot result.

## Prove update-aware signed PCR 11 authorization

A no-change regeneration can produce an identical reproducible UKI and does
not prove that the token accepts a *new* PCR 11 value. Use one temporary,
explicit default command-line parameter to make a measured input change
without changing the intended filesystem-check policy.

First preserve the complete working normal command line without overwriting a
previous test:

```bash
test ! -e /etc/kernel/cmdline.before-signed-pcr-test
sudo cp -a /etc/kernel/cmdline /etc/kernel/cmdline.before-signed-pcr-test
sudo sha256sum /boot/EFI/Linux/arch-linux.efi
systemd-analyze pcrs 11
```

Open `/etc/kernel/cmdline` and append exactly one parameter:

```bash
sudo micro /etc/kernel/cmdline
```

```text
fsck.mode=auto
```

`auto` is systemd-fsck's default. It is only a temporary measured marker; do
not use `force` or `skip`. Preserve the real UUID, combined TPM option,
`zswap.enabled=0`, root mapper, `rw`, `quiet`, and `splash`.

Review, rebuild, and inspect without reenrolling:

```bash
cat /etc/kernel/cmdline
grep -F 'fsck.mode=auto' /etc/kernel/cmdline
! grep -F 'fsck.mode=auto' /etc/kernel/cmdline-fallback
sudo mkinitcpio -P
sudo sha256sum /boot/EFI/Linux/arch-linux.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

The normal UKI hash and expected PCR 11 signature must change, while the
fallback command line remains untouched. Do not run `systemd-cryptenroll`.

Cold boot the changed normal UKI and enter the same TPM PIN. A successful
unlock proves that the token accepted a newly signed PCR 11 value instead of
one raw value fixed at enrollment. After login:

```bash
grep -F 'fsck.mode=auto' /proc/cmdline
systemd-analyze pcrs 11
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo sbctl verify
systemctl --failed --no-pager
```

Exactly one TPM2 enrollment must remain. Restore the normal source
byte-for-byte, regenerate its new signed policy, and verify both UKIs:

```bash
sudo cp -a /etc/kernel/cmdline.before-signed-pcr-test /etc/kernel/cmdline
! grep -F 'fsck.mode=auto' /etc/kernel/cmdline
sudo mkinitcpio -P
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

Cold boot normal once more and prove the TPM PIN works with the restored
production command line. A changed UKI that requires reenrollment indicates a
missing or invalid `.pcrsig`, a different public key, or another policy mistake
and blocks completion.

## Update lifecycle after enrollment

For every transaction that changes the kernel, microcode, systemd, ukify,
mkinitcpio, the initramfs, Plymouth, its theme, or an embedded command line:

```bash
sudo mkinitcpio -P
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux-fallback.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

Do not reboot if either UKI lacks its PCR sections, uses the wrong command
line, fails to build, or is unsigned. A new valid signed PCR 11 value is
accepted by the existing token; ordinary UKI updates do not require
reenrollment.

Raw PCR 7 is different. Changing Secure Boot enablement, PK, KEK, db, dbx, or
the validating authorities can make TPM release fail. After an intentional
trust-policy change:

1. use the strong LUKS passphrase;
2. explain and verify the new Secure Boot state;
3. verify both UKIs and their PCR-policy sections;
4. prove password and recovery slots remain valid;
5. reenroll with the reviewed command while wiping only old TPM2 slots;
6. refresh the dated LUKS header backup;
7. retest normal PIN and fallback passphrase paths.

Firmware updates do not justify clearing the TPM. Have the fallback,
passphrase, recovery key, and ISO available; accept a manual prompt after the
update; then inspect TPM discovery, PCR 7, the event log, Secure Boot, and UKIs
before considering reenrollment.

## Diagnosis and safe recovery

| Symptom | Likely boundary | Safe first action |
| --- | --- | --- |
| Normal asks for full LUKS passphrase | TPM request was skipped or its policy refused the state | Use the passphrase, then inspect; do not weaken policy blindly |
| No TPM PIN appears | Normal command line, token discovery, or TPM userspace | Use passphrase and inspect `tpm2-device=auto`, token inventory, and TPM discovery |
| PIN is accepted but unlock falls back | PCR 7, signed PCR 11, public key, signature, or TPM state | Use passphrase and compare the current boot before reenrolling |
| TPM stopped working after an ordinary UKI update | Missing/invalid `.pcrsig` or wrong PCR-policy key | Passphrase boot, repair the UKI build, and keep the token unchanged |
| TPM stopped working after Secure Boot policy change | Raw PCR 7 changed | Verify the intended trust change, then perform controlled reenrollment |
| Repeated PIN attempts caused a delay | TPM dictionary-attack lockout | Use passphrase; wait and inspect deliberately; never clear TPM as first aid |
| Normal fails but fallback accepts passphrase | TPM, signed policy, Plymouth, or normal-only command line | Stay on trusted fallback and compare artifacts |
| Both UKIs reject a known passphrase | Keyboard, wrong device, header, or credential problem | Stop retries and use ISO read-only inspection |

Useful evidence after a passphrase recovery boot is:

```bash
bootctl --esp-path=/boot status
cat /proc/cmdline
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo ukify --section=.pcrsig:text --section=.pcrpkey:text inspect \
  /boot/EFI/Linux/arch-linux.efi
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
systemd-analyze has-tpm2
systemd-analyze pcrs 7 11
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup luksDump /dev/nvme0n1p2
sudo sbctl status
sudo sbctl verify
journalctl -b -u 'systemd-cryptsetup@cryptlvm.service' --no-pager
```

### Enrollment reports `Failed to unseal secret using TPM2: Device not a stream`

Stop and inspect the LUKS inventory; do not assume that a usable token was
created:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup luksDump /dev/nvme0n1p2
```

During the 2026-09-06 validation, this occurred when the UKI signed only the
`enter-initrd` PCR 11 state but enrollment was being performed after the
running system had reached later boot phases. Keep the four-value `Phases=`
line documented above, rebuild both UKIs, inspect their PCR sections, and cold
boot the rebuilt normal UKI. Only after that reboot, verify that the runtime
signature exists and the public-key hashes match:

```bash
sudo test -s /run/systemd/tpm2-pcr-signature.json
sudo test -s /run/systemd/tpm2-pcr-public-key.pem
sudo sha256sum \
  /run/systemd/tpm2-pcr-public-key.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem
```

The configuration change does not require generating a new key pair. Do not
delete or replace the existing PCR key files merely to add phase signatures.
Retry enrollment only after TPM discovery, PCR access, both UKI signatures,
and the manual LUKS passphrase remain healthy.

If an earlier revision of this chapter already created the original
`pcr-policy` backup with only `Phases=enter-initrd`, preserve it. Reopen and
mount `ARCH-BACKUP` with the complete persistent-device procedure in **Create
the before-TPM recovery checkpoint**, then create a separate, non-overwriting
checkpoint for the corrected configuration and the same key pair:

```bash
tpm_backup_root="$backup_mount/rogue-thinkpad-recovery"
tpm_correction="$tpm_backup_root/post-install-20-after-phase-policy-correction"
test -d "$tpm_backup_root"
test ! -e "$tpm_correction"
sudo install -d -m 0700 "$tpm_correction"
sudo cp --archive \
  /etc/kernel/uki.conf \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem \
  "$tpm_correction/"
sudo sh -c 'cd "$1" && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS' sh "$tpm_correction"
sudo chmod -R go-rwx "$tpm_correction"
sudo chown -R root:root "$tpm_correction"
sudo sh -c 'cd "$1" && sha256sum --check SHA256SUMS' sh "$tpm_correction"
sync
udisksctl unmount -b "$backup_mapping"
udisksctl lock -b "$backup_partition"
udisksctl power-off -b "$(readlink -f "$backup_disk")"
```

All closing operations must succeed before disconnecting the disk. This
checkpoint is only for a machine that already followed the superseded
single-phase instructions; a fresh run uses the normal before-TPM checkpoint.

### Enrollment rejects `--tpm2-public-key-pcrs=11:sha256`

The systemd build on this machine reports:

```text
Not expecting hash algorithm specification in PCR mask value, refusing: 11:sha256
```

Use `--tpm2-public-key-pcrs=11` exactly as shown by the corrected enrollment
command. Keep `--tpm2-pcrs=7:sha256` unchanged: the first is a signed-PCR
mask, while the second explicitly selects the bank for a raw current value.
The SHA-256 bank for the signed policy remains declared by
`PCRBanks=sha256` and can be confirmed in the UKI's `.pcrsig` output.

### Forgotten TPM PIN

Use the strong passphrase or recovery key. Do not guess repeatedly and do not
clear the TPM. From a verified normal boot, confirm both manual credentials,
the intended PCR policy, and the current signed UKIs; then run the enrollment
command with a new unique PIN. `--wipe-slot=tpm2` replaces only the previous
TPM2 enrollment after the new one succeeds.

### Missing policy signature after an update

Continue with the strong passphrase. Inspect `/etc/kernel/uki.conf`, key paths
and modes, both `.pcrsig`/`.pcrpkey` sections, the runtime files, mkinitcpio
output, and sbctl verification. Repair the build source and rebuild both UKIs.
Do not reenroll a token against an incomplete or unexplained policy.

### TPM disabled, cleared, or replaced with the motherboard

Use the textual fallback and a strong LUKS credential. The manual LUKS slots
remain independent of the missing TPM. After the replacement platform has a
known-good Secure Boot and UKI state, audit its TPM and perform a fresh
enrollment. Reformatting LUKS or restoring an old header is not required.

### Neither UKI reaches a usable credential prompt

Boot the verified Arch ISO in UEFI mode. Identify `/dev/nvme0n1p2`, unlock it
with the strong passphrase or recovery key, activate `vg0`, mount root, home,
and the ESP at the chapter 12 targets, and inspect before editing. Restore the
normal manual command line or rebuild known-good signed UKIs, then test the
textual fallback first.

A TPM failure does not justify `luksFormat`, `pvcreate`, `vgcreate`, filesystem
creation, LUKS header restoration, Secure Boot key deletion, firmware factory
key restoration, or system reinstallation.

## Roll back to passphrase-only boot

Rollback first removes the TPM request from boot, proves manual unlock, and
only then removes the extra LUKS credential.

1. Boot the textual fallback and prove the strong passphrase works.
2. Restore only the saved pre-TPM normal command line:

```bash
sudo cp -a /etc/kernel/cmdline.before-tpm2 /etc/kernel/cmdline
cat /etc/kernel/cmdline
cat /etc/kernel/cmdline-fallback
```

3. Rebuild, inspect, and verify both UKIs:

```bash
sudo mkinitcpio -P
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

4. Cold boot the normal UKI and prove it asks for and accepts the strong LUKS
   passphrase.
5. Test the passphrase and recovery key again with `cryptsetup open
   --test-passphrase`.
6. Only after both manual routes work, remove TPM2 enrollments:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2 --wipe-slot=tpm2
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo cryptsetup open --test-passphrase /dev/nvme0n1p2
```

This removes only TPM2 LUKS slots and tokens. It does not clear the physical
TPM or touch password and recovery slots.

The PCR sections may remain harmlessly in UKIs. For a complete rollback,
archive their configuration and keys on encrypted root instead of deleting
them immediately:

```bash
tpm_retired_dir="/root/tpm2-policy-retired-$(date +%Y%m%d-%H%M%S)"
sudo install -d -m 0700 "$tpm_retired_dir"
sudo mv \
  /etc/kernel/uki.conf \
  /etc/systemd/tpm2-pcr-private-key-initrd.pem \
  /etc/systemd/tpm2-pcr-public-key-initrd.pem \
  "$tpm_retired_dir/"
sudo mkinitcpio -P
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
sudo sbctl verify
```

Cold boot normal and fallback again, then create a new dated LUKS header and
configuration checkpoint on encrypted external media. Do not restore the old
header merely to remove a functioning token.

## Completion checklist

- [ ] Chapter 19 is recorded as hardware-validated on 2026-09-05.
- [ ] The normal Plymouth and textual fallback UKIs were retested before TPM work.
- [ ] One intended TPM2 is discoverable and PCR 7 plus PCR 11 are readable in SHA-256.
- [ ] `tpm2-tss` is installed through a complete system upgrade.
- [ ] The real LUKS UUID matches both command-line sources.
- [ ] The strong LUKS passphrase remains present and passes `--test-passphrase`.
- [ ] A generated recovery key is stored offline and passes `--test-passphrase`.
- [ ] Distinct before-enrollment LUKS2 header backups exist on encrypted external storage.
- [ ] A unique per-ThinkPad PCR-policy key pair exists outside Git and the ESP.
- [ ] The PCR private key is root-only and backed up on encrypted recovery media.
- [ ] Both UKIs contain valid `.pcrsig` and `.pcrpkey` sections for SHA-256 PCR 11.
- [ ] The runtime PCR public key matches the configured public key.
- [ ] Only the normal command line requests `discard,tpm2-device=auto`.
- [ ] Fallback remains textual and omits TPM, `quiet`, and `splash`.
- [ ] Exactly one TPM2 token requires a unique PIN and binds raw PCR 7 plus signed PCR 11.
- [ ] Password and recovery slots were never wiped.
- [ ] A distinct after-enrollment LUKS2 header backup exists on encrypted external storage.
- [ ] Normal cold boot unlocks with the TPM PIN and reaches the healthy workstation.
- [ ] Textual fallback unlocks with the strong LUKS passphrase without using TPM.
- [ ] A temporary `fsck.mode=auto` marker changes the normal UKI measurement,
      unlocks through the same token without reenrollment, and is then removed.
- [ ] Both UKIs and every executed bootloader artifact verify under sbctl.
- [ ] No PIN, recovery key, private PCR key, LUKS header, or token dump entered Git.
- [ ] `post-install-18-v2` remains the latest required dotfiles checkpoint.
- [ ] Clearing the TPM, unattended TPM unlock, raw PCR 11 binding, hibernation, and dotfiles changes remain absent.

This complete sequence passed hardware validation on the target ThinkPad T14
Gen 1 AMD on 2026-09-06. Normal boot unlocks through TPM2 plus the unique PIN,
the textual fallback and manual credentials remain independent, and a changed
signed-PCR UKI continues to unlock through the same token without reenrollment.

## Sources

- [Arch package: tpm2-tss](https://archlinux.org/packages/core/x86_64/tpm2-tss/)
- [ArchWiki: Trusted Platform Module](https://wiki.archlinux.org/title/Trusted_Platform_Module)
- [ArchWiki: systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll)
- [`systemd-cryptenroll(1)`](https://man.archlinux.org/man/systemd-cryptenroll.1.en)
- [`systemd-cryptsetup(8)`](https://man.archlinux.org/man/systemd-cryptsetup.8.en)
- [`systemd-cryptsetup-generator(8)`](https://man.archlinux.org/man/systemd-cryptsetup-generator.8.en)
- [`crypttab(5)`](https://man.archlinux.org/man/crypttab.5.en)
- [`ukify(1)`](https://man.archlinux.org/man/ukify.1.en)
- [`systemd-stub(7)`](https://man.archlinux.org/man/systemd-stub.7.en)
- [`systemd-fsck@.service(8)`](https://man.archlinux.org/man/systemd-fsck@.service.8.en)
- [`mkinitcpio(8)`](https://man.archlinux.org/man/mkinitcpio.8.en)
- [`cryptsetup-open(8)`](https://man.archlinux.org/man/cryptsetup-open.8.en)
- [`cryptsetup-luksHeaderBackup(8)`](https://man.archlinux.org/man/cryptsetup-luksHeaderBackup.8.en)
- [systemd TPM2 PCR measurement registry](https://systemd.io/TPM2_PCR_MEASUREMENTS/)

## Next step

Continue with chapter 21 to record the pre-personalization checkpoint and
refine the existing Waybar without changing the now validated TPM policy.
Treat any future Secure Boot, PCR-policy key, LUKS token, or early-boot change
as a separate security-sensitive extension with its own recovery gate.
