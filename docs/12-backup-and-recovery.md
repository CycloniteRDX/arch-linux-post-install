# 12 — Establish backup and recovery

## Goal

Make the workstation recoverable from both ordinary data loss and boot-chain
failure. This chapter establishes four distinct safeguards:

1. an offline recovery bundle containing the LUKS2 header, sbctl state, Secure
   Boot keys, and a non-secret system inventory;
2. an encrypted Restic repository for `/home/neon` on external storage;
3. a real test restore into an empty directory;
4. a read-only Arch ISO recovery rehearsal through LUKS, LVM, mounts, and
   `arch-chroot`.

The chapter does not automate retention, connect cloud storage, rotate LUKS or
Secure Boot keys, restore a LUKS header, repartition a disk, or reinstall the
bootloader. Those operations require a separately reviewed recovery event.

## Canonical storage map

Every command in this chapter assumes the installed profile:

| Layer | Canonical object |
| --- | --- |
| EFI System Partition | `/dev/nvme0n1p1`, FAT32, mounted at `/boot` |
| LUKS2 container | `/dev/nvme0n1p2` |
| Open mapping | `/dev/mapper/cryptlvm` |
| Volume group | `vg0` |
| Root | `/dev/vg0/root`, ext4, mounted at `/` |
| Home | `/dev/vg0/home`, ext4, mounted at `/home` |
| Disk swap | `/dev/vg0/swap`, 16 GiB |
| Normal UKI | `/boot/EFI/Linux/arch-linux.efi` |
| Fallback UKI | `/boot/EFI/Linux/arch-linux-fallback.efi` |
| sbctl state and private keys | `/var/lib/sbctl` |

Stop if the live system does not match this table. Never substitute a device
name from memory when writing recovery metadata.

## Threat and backup model

The minimum target is the 3-2-1 principle: three copies of important data, on
two storage types, with one copy stored separately. This chapter creates the
first external copy and defines the second; it cannot make two physical disks
appear automatically.

The recovery bundle and user-data repository have different purposes:

| Backup | Contains | Sensitivity | Update trigger |
| --- | --- | --- | --- |
| Recovery bundle | LUKS header, sbctl private keys/state, EFI variable exports, inventory | Critical secrets; can assist decryption or boot signing | After LUKS keyslot/token or Secure Boot key changes |
| Restic repository | Encrypted snapshots of `/home/neon` | Personal data; encrypted by Restic | Regularly and before risky work |
| Git remotes | Public project history and dotfiles | No secrets | On each reviewed commit |

Git is not a backup of uncommitted work, browser profiles, application state,
photos, documents, or credentials. LUKS encryption protects a stolen internal
SSD; it does not protect against deletion, filesystem damage, or SSD failure.

## Prerequisites and external-media checkpoint

- Chapters 01 through 11 are complete.
- The machine boots both UKIs with Secure Boot enabled.
- The current LUKS passphrase is known and has been tested recently.
- A separate encrypted external disk with enough free space is mounted
  read-write. The filesystem used for the raw recovery bundle must preserve
  Unix ownership and permissions.
- A second secure location is available for another copy of the small recovery
  bundle and the Restic password.

Connect the intended backup disk and identify it without changing anything:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
findmnt --real
```

This guide uses the placeholder mount point:

```text
/run/media/neon/ARCH-BACKUP
```

Replace it in every command with the exact mount point printed by `findmnt`.
Do not continue merely because a similarly named directory exists. Confirm the
path is a mounted external filesystem:

```bash
findmnt --target /run/media/neon/ARCH-BACKUP
df -hT /run/media/neon/ARCH-BACKUP
```

The source must be the external disk. If `findmnt` resolves to `/`, `/home`, or
the internal NVMe, stop.

For the raw recovery bundle, also stop if the external filesystem is exFAT,
FAT32, or NTFS, or if the underlying external device is not encrypted. Restic
encrypts its own repository and can use many filesystems, but the raw LUKS and
sbctl material created below relies on an encrypted filesystem that preserves
root-only permissions. Preparing the external disk itself is intentionally
outside this chapter because formatting it is destructive.

## Install the backup tools

Read Arch News and perform one complete update transaction:

```bash
sudo pacman -Syu restic efitools
```

`cryptsetup`, `lvm2`, `sbctl`, `arch-install-scripts`, Git, and OpenSSH are
already present. Verify the required commands:

```bash
command -v restic cryptsetup sbctl efi-readvar arch-chroot
restic version
```

## Audit the live system before copying secrets

```bash
lsblk -f
findmnt /
findmnt /home
findmnt /boot
swapon --show
sudo cryptsetup luksDump /dev/nvme0n1p2
sudo sbctl status
sudo sbctl verify
sudo ls -la /var/lib/sbctl
```

The LUKS output contains metadata but not the plaintext passphrase. Still, do
not post the complete output publicly because token and keyslot information can
reveal system details.

Confirm `/var/lib/sbctl/keys` exists and is root-owned. Exporting only firmware
variables is insufficient: PK, KEK, db, and dbx exports are public enrolled
data, while `/var/lib/sbctl` contains the private signing keys required to keep
signing future UKIs with the existing trust chain.

## Create the offline recovery bundle

Create a root-only directory directly on the verified external disk:

```bash
sudo install -d -m 0700 /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
sudo install -d -m 0700 /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/secure-boot-variables
```

Back up the LUKS2 header and keyslot area:

```bash
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/nvme0n1p2-luks2-header.img
```

This binary file is highly sensitive. A header backup plus a passphrase valid
at the time of the backup may retain access even after later keyslot changes.
Protect it like a credential and update it after intentional LUKS key changes.

Copy the complete sbctl state while preserving ownership and modes:

```bash
sudo cp --archive /var/lib/sbctl /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/sbctl
```

Export the enrolled UEFI Secure Boot variables for diagnosis and manual
firmware recovery:

```bash
sudo efi-readvar -v PK  -o /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/secure-boot-variables/PK.esl
sudo efi-readvar -v KEK -o /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/secure-boot-variables/KEK.esl
sudo efi-readvar -v db  -o /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/secure-boot-variables/db.esl
sudo efi-readvar -v dbx -o /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/secure-boot-variables/dbx.esl
```

Copy the small boot and system configuration set. These copies support
diagnosis; they do not replace a data backup:

```bash
sudo install -d -m 0700 /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/system-config
sudo cp --archive /etc/fstab /etc/kernel/cmdline /etc/mkinitcpio.conf /etc/mkinitcpio.d /etc/pacman.conf /etc/greetd /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/system-config/
sudo cp --archive /boot/loader /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/system-config/
```

If `/etc/crypttab.initramfs` exists, copy it separately:

```bash
sudo cp --archive /etc/crypttab.initramfs /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/system-config/
```

If it is absent, confirm the UKI uses `rd.luks.name=` from
`/etc/kernel/cmdline` and continue. Do not use a wildcard around `/etc` or
`/boot`.

Generate a non-secret inventory on the external disk:

```bash
sudo sh -c 'lsblk -f > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/lsblk.txt'
sudo sh -c 'blkid > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/blkid.txt'
sudo sh -c 'pvs > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/pvs.txt'
sudo sh -c 'vgs > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/vgs.txt'
sudo sh -c 'lvs -a -o +devices > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/lvs.txt'
sudo sh -c 'pacman -Qqe > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/packages-explicit.txt'
sudo sh -c 'systemctl list-unit-files --state=enabled > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/enabled-system-units.txt'
sudo sh -c 'sbctl status > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/sbctl-status.txt'
sudo sh -c 'sbctl list-files > /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/sbctl-files.txt'
```

Record checksums and restrict the complete bundle:

```bash
sudo sh -c 'cd /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery && find . -type f ! -name SHA256SUMS -print0 | sort -z | xargs -0 sha256sum > SHA256SUMS'
sudo chmod -R go-rwx /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
sudo chown -R root:root /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery
```

Do not copy this directory into Git, cloud notes, ChatGPT, email, or an
unencrypted everyday USB stick. If the external filesystem itself is not
encrypted, place this recovery bundle inside a separately encrypted container
before transporting or storing it.

## Verify the recovery bundle without restoring it

Check file types, ownership, sizes, and checksums:

```bash
sudo find /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery -maxdepth 3 -printf '%M %u:%g %s %p\n'
sudo file /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/nvme0n1p2-luks2-header.img
sudo sh -c 'cd /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery && sha256sum --check SHA256SUMS'
```

Inspect the backup header as a file, not by writing it to the live partition:

```bash
sudo cryptsetup luksDump /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/nvme0n1p2-luks2-header.img
```

It must report a LUKS2 header with the expected UUID. Compare UUIDs:

```bash
sudo cryptsetup luksUUID /dev/nvme0n1p2
sudo cryptsetup luksUUID /run/media/neon/ARCH-BACKUP/rogue-thinkpad-recovery/nvme0n1p2-luks2-header.img
```

The two values must match. Never run `luksHeaderRestore` merely as a test: it
writes critical metadata to the target device.

## Initialize the encrypted Restic repository

Choose a unique, long Restic password and store it separately from the external
disk. Losing every Restic password makes the repository unrecoverable. Anyone
with the repository and a valid password can read its contents.

Create the repository interactively so the password is not exposed through
shell history, a process list, or this repository:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad init
```

Do not use `--insecure-no-password`, `RESTIC_PASSWORD`, or a plaintext password
file in this manual baseline.

## Define the first user-data snapshot

Back up the complete user home while excluding only reproducible caches and
Trash:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad backup /home/neon --one-file-system --exclude '/home/neon/.cache' --exclude '/home/neon/.local/share/Trash' --tag manual-baseline
```

Review those exclusions against actual data before accepting them. Downloads
are included in the first baseline; future exclusions must be deliberate and
documented. Restic reads files as `neon`; files in the home directory that are
unreadable by the owner will be reported as errors and must not be silently
ignored.

Inspect the resulting snapshot:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad snapshots
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad stats latest
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad check --read-data-subset=5%
```

`check` validates repository structure and samples stored data. A later
maintenance policy can schedule full reads periodically; the initial restore
test below proves that useful files can actually be extracted.

## Perform a real test restore

Create a new empty directory on the internal home filesystem:

```bash
mkdir -m 0700 /home/neon/restic-restore-test
```

Select one known, non-secret directory or file from the snapshot:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad ls latest
```

Restore a small reviewed path. The following example uses the dotfiles clone;
replace the include path if the repository lives elsewhere:

```bash
restic --repo /run/media/neon/ARCH-BACKUP/restic-rogue-thinkpad restore latest --target /home/neon/restic-restore-test --include /home/neon/Projects/CycloniteRDX/niri-dotfiles
```

Compare the restored tree with the live source:

```bash
diff --recursive --brief /home/neon/Projects/CycloniteRDX/niri-dotfiles /home/neon/restic-restore-test/home/neon/Projects/CycloniteRDX/niri-dotfiles
```

No output means the compared file contents match. After reviewing the restored
copy, remove the test directory through the graphical file manager so the
operation is recoverable from Trash. Do not add the restored copy to Git.

## Create the second critical copy

Copy `rogue-thinkpad-recovery` to a second encrypted storage device kept in a
different physical location. Verify its `SHA256SUMS` on that device. Store the
Restic password independently, for example in a password manager with a tested
recovery method or a sealed offline record.

The second copy does not need to duplicate the full Restic repository on day
one, but truly irreplaceable user data still needs an off-site copy. A later
chapter or handbook guide will compare a second Restic disk, SFTP/NAS, and
supported object-storage backends before automating uploads.

## Prepare the Arch installation medium

Use a current verified Arch ISO on a dedicated USB drive. Keep it physically
available; an ISO file stored only inside the encrypted laptop is useless when
the laptop cannot boot. Record neither Wi-Fi credentials nor the LUKS
passphrase on the USB.

This rehearsal is read-only with respect to installed files. Rebooting is
required, so save work and keep the recovery bundle disconnected until the ISO
has booted.

## Rehearse ISO unlock, mount, and chroot

Boot the Arch ISO in UEFI mode. If custom Secure Boot rejects the ISO,
temporarily disable Secure Boot in firmware; do not clear enrolled keys or
enable Setup Mode.

Confirm UEFI and identify the disk:

```bash
cat /sys/firmware/efi/fw_platform_size
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL
```

The first command should print `64`. Verify the target is the internal NVMe,
then unlock the existing LUKS container:

```bash
cryptsetup open /dev/nvme0n1p2 cryptlvm
cryptsetup status cryptlvm
```

Activate LVM and inspect it:

```bash
vgchange --activate y vg0
lvs -a -o +devices
```

Mount the installed system without modifying boot configuration:

```bash
mount /dev/vg0/root /mnt
mount --mkdir /dev/vg0/home /mnt/home
mount --mkdir -o fmask=0177,dmask=0077 /dev/nvme0n1p1 /mnt/boot
findmnt /mnt /mnt/home /mnt/boot
```

Enter the installation:

```bash
arch-chroot /mnt
```

Inside the chroot, perform read-only checks:

```bash
cat /etc/fstab
cat /etc/kernel/cmdline
ls -lh /boot/EFI/Linux
bootctl --esp-path=/boot list
sbctl status
sbctl verify
grep -v '^[[:space:]]*#' /etc/mkinitcpio.conf
ls -l /etc/mkinitcpio.d
```

Do not reinstall systemd-boot, regenerate keys, enroll firmware variables,
restore the LUKS header, rebuild UKIs, or update packages during this rehearsal.
The goal is to prove access and understand the recovery boundary.

Exit and unmount cleanly:

```bash
exit
umount -R /mnt
vgchange --activate n vg0
cryptsetup close cryptlvm
lsblk
reboot
```

Remove the ISO USB when firmware begins rebooting and re-enable Secure Boot if
it was temporarily disabled. Confirm the normal signed UKI still boots.

## Recovery decision table

| Failure | First recovery action | Do not do first |
| --- | --- | --- |
| User file deleted | Restore the selected path from Restic to a separate target | Restore the whole home over the live filesystem |
| Restic repository error | Run `restic check`, preserve the disk, and diagnose | Run prune or repair blindly |
| Normal UKI fails | Select `arch-linux-fallback.efi` in systemd-boot | Repartition or restore LUKS metadata |
| Both UKIs fail | Boot ISO, unlock, mount, chroot, inspect mkinitcpio and signatures | Generate new Secure Boot keys |
| Secure Boot violation | Disable Secure Boot temporarily and inspect signatures from ISO/chroot | Clear firmware keys |
| LUKS passphrase rejected | Check keyboard layout and available keyslots from ISO | Restore an old header immediately |
| LUKS header damaged | Preserve the disk, verify device identity and backup UUID, plan a reviewed header restore | Guess the device or overwrite it while diagnosing |
| Internal SSD failure | Replace storage, reinstall from runbook, restore user data and only then restore necessary identity material | Clone writes onto the failed source disk |

## Completion checklist

- [ ] External storage was identified through `lsblk` and `findmnt`.
- [ ] The LUKS header backup exists and its UUID matches the live container.
- [ ] `/var/lib/sbctl` and EFI variable exports exist in the offline bundle.
- [ ] Every recovery-bundle checksum verifies.
- [ ] The critical bundle has a second encrypted, physically separate copy.
- [ ] The Restic password is stored separately and recoverably.
- [ ] The first `/home/neon` snapshot completed without ignored errors.
- [ ] `restic check` succeeds.
- [ ] A selected path was restored and compared successfully.
- [ ] The Arch ISO can unlock LUKS, activate LVM, mount all filesystems, and enter the chroot.
- [ ] The rehearsal made no writes to LUKS headers or Secure Boot keys.
- [ ] The installed system still boots normally with Secure Boot enabled.

## Sources

- [cryptsetup header backup manual](https://man.archlinux.org/man/cryptsetup-luksHeaderBackup.8.en)
- [cryptsetup manual](https://man.archlinux.org/man/cryptsetup.8.en)
- [Restic documentation](https://restic.readthedocs.io/en/latest/)
- [Restic package](https://archlinux.org/packages/extra/x86_64/restic/)
- [sbctl upstream](https://github.com/Foxboron/sbctl)
- [arch-chroot manual](https://man.archlinux.org/man/arch-chroot.8.en)
