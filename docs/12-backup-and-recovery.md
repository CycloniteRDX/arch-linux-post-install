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
Secure Boot keys, restore a LUKS header, repartition the internal system disk,
or reinstall the bootloader. It can prepare one dedicated external backup disk
after an explicit destructive checkpoint. Shrinking a filesystem, preserving
files already on that disk, and restoring old backup media remain separate
operations.

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
| External backup | Dedicated GPT disk with one LUKS2 partition and ext4 filesystem labelled `ARCH-BACKUP` |

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
- A dedicated external disk with enough capacity is available. It may already
  use the canonical encrypted layout, or it may be an expendable disk that can
  be erased and prepared below. An NTFS disk is not ready for the complete
  chapter merely because Restic could store repository files on it.
- A second secure location is available for another copy of the small recovery
  bundle and the Restic password.

Confirm that the storage tools inherited from earlier chapters are present:

```bash
command -v lsblk findmnt fdisk wipefs udevadm cryptsetup mkfs.ext4 udisksctl lsof
```

Every command name must resolve before preparing the disk.

Connect the intended backup disk and identify it without changing anything:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
findmnt --real
```

The external disk must be distinguishable from the internal NVMe by size,
model, serial number and `TRAN=usb`. If it already contains data, stop here.
Copy that data to another device and verify the copy before using any command
from the destructive preparation section. Moving the only copy elsewhere on
the same disk does not protect it from repartitioning.

## Prepare a dedicated external backup disk

Resolve and verify the persistent device path below for every disk. When
`lsblk` already shows the intended external disk as one LUKS2 partition
containing an ext4 filesystem, skip only the destructive subsections from
**Unmount the existing NTFS filesystem** through **Establish the mount-root
permissions**, then continue at **Test desktop unlocking and mounting**. The
canonical layout is:

```text
external disk
└─ GPT partition 1 — LUKS2 label ARCH-BACKUP-LUKS
   └─ dm-crypt mapping
      └─ ext4 label ARCH-BACKUP
```

LUKS protects the whole external filesystem, including the raw LUKS-header and
Secure Boot material. Restic still encrypts its own repository independently:
the two layers protect different objects and use independently recoverable
passwords. The disk remains offline and locked when no backup or recovery task
is running; do not add it to `/etc/fstab` or `/etc/crypttab`.

This layout dedicates the disk to Linux backup and recovery. Windows does not
natively unlock LUKS2 or mount ext4. If the physical disk must continue serving
as ordinary Windows storage, use a different dedicated backup disk rather than
weakening or complicating the recovery layout.

### Resolve one persistent device path

List the persistent names for USB storage:

```bash
ls -l /dev/disk/by-id/usb-*
```

Choose the entry for the whole backup disk, not an entry ending in `-part1`,
and assign its exact path. The following value is deliberately invalid and
must be replaced:

```bash
backup_disk=/dev/disk/by-id/usb-REPLACE_WITH_THE_EXACT_WHOLE_DISK_ID
backup_partition="${backup_disk}-part1"
```

These shell variables are not persistent. Opening a new terminal, logging out,
or rebooting discards them. Repeat both assignments and the checks below before
continuing in a new shell; never reconstruct the target from a remembered
kernel name such as `/dev/sda`.

Resolve and inspect the selection without modifying it:

```bash
test -b "$backup_disk"
readlink -f "$backup_disk"
lsblk -d -o NAME,PATH,SIZE,TYPE,MODEL,SERIAL,TRAN,RM "$(readlink -f "$backup_disk")"
sudo fdisk --list "$backup_disk"
```

Required observations:

- `readlink` resolves to the external disk, for example `/dev/sda`, and never
  to `/dev/nvme0n1`;
- the model, serial number and capacity match the physical backup disk;
- `TYPE` is `disk` and `TRAN` is `usb`;
- `fdisk` shows the NTFS partition expected on that same disk.

Do not continue when any property is ambiguous. A kernel name such as
`/dev/sda` can change between boots; the persistent `/dev/disk/by-id/` path is
kept in the shell variable so later commands cannot silently select a different
disk.

### Unmount the existing NTFS filesystem

Inspect every child and mount point again:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS "$(readlink -f "$backup_disk")"
```

If the NTFS partition is mounted, close every application using it and unmount
that exact partition. For a disk with one existing partition:

```bash
udisksctl unmount -b "$(readlink -f "$backup_partition")"
```

If the persistent `-part1` name does not exist, use the exact child path shown
by `lsblk`. Confirm that no child of the selected disk has a mount point before
continuing.

### Destructive checkpoint

The following operation erases the partition table and makes the existing NTFS
contents inaccessible. It is not an in-place conversion. Run the two inspection
commands one final time and physically compare their model, serial and capacity
with the intended disk:

```bash
lsblk -d -o NAME,PATH,SIZE,MODEL,SERIAL,TRAN "$(readlink -f "$backup_disk")"
sudo fdisk --list "$backup_disk"
```

Only after the old contents exist elsewhere and the target is certain, open
`fdisk` with an exclusive lock:

```bash
sudo fdisk --lock=yes "$backup_disk"
```

Inside `fdisk`, enter these commands one at a time:

```text
p       inspect the existing table one last time
g       create a new empty GPT
n       create partition 1
Enter   accept partition number 1
Enter   accept the aligned first sector
Enter   use the remaining disk capacity
p       inspect the proposed GPT
w       write it and exit
```

Nothing is written before `w`. If the displayed disk is wrong, use `q` instead
and stop. After writing, let udev settle and verify that the persistent
partition path now exists:

```bash
sudo udevadm settle
test -b "$backup_partition"
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS "$(readlink -f "$backup_disk")"
```

### Create LUKS2 and ext4

Remove any stale filesystem signature from the new partition, then create a
labelled LUKS2 container. Both commands below are destructive and must target
partition 1, never the whole disk or the internal NVMe:

```bash
sudo wipefs --all "$backup_partition"
sudo cryptsetup luksFormat --type luks2 --verify-passphrase --label ARCH-BACKUP-LUKS "$backup_partition"
```

Type the explicit confirmation requested by `cryptsetup` and choose a long,
unique passphrase. Store its recovery copy separately from this disk. Do not
reuse either the laptop LUKS passphrase or the future Restic password.

The desktop may display a password prompt as soon as udev detects the new LUKS
container. This is expected removable-media behaviour. Cancel the prompt while
following the manual preparation below. If the password was already entered,
the desktop may have created a dm-crypt mapping automatically. Inspect the
partition:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS "$backup_partition"
```

If a `crypt` child is present, close that automatic mapping before creating the
manual mapping:

```bash
udisksctl lock -b "$backup_partition"
```

Run the `lsblk` command again and confirm that the `crypt` child has
disappeared. Do not open the same LUKS container simultaneously under two
different mapping names.

Open the new container and create ext4 inside it:

```bash
sudo cryptsetup open "$backup_partition" arch-backup
sudo mkfs.ext4 -L ARCH-BACKUP /dev/mapper/arch-backup
```

Creating the ext4 signature may trigger another graphical password prompt.
Cancel it: `/dev/mapper/arch-backup` is already the active manual mapping and a
second desktop unlock is neither required nor useful at this stage.

Do not pass hand-chosen cipher, key-size, PBKDF, sector-size or ext4 tuning
options. The current tools select hardware-aware and maintained defaults; this
chapter has no measured requirement that justifies overriding them.

### Establish the mount-root permissions

Mount the empty filesystem once and make its root writable only by the
canonical user. Root can still create the protected recovery directory later:

```bash
sudo install -d -m 0700 /mnt/arch-backup-setup
sudo mount /dev/mapper/arch-backup /mnt/arch-backup-setup
sudo chown neon:neon /mnt/arch-backup-setup
sudo chmod 0700 /mnt/arch-backup-setup
findmnt --target /mnt/arch-backup-setup
stat -c '%A %U:%G %n' /mnt/arch-backup-setup
df -hT /mnt/arch-backup-setup
sudo umount /mnt/arch-backup-setup
sudo cryptsetup close arch-backup
```

The expected permission line begins with `drwx------ neon:neon`. That ownership
is stored in ext4 and therefore remains correct when the disk is mounted again.

### Test desktop unlocking and mounting

The manual `arch-backup` mapping was closed above. The normal desktop path now
has three separate objects:

| Object | Example | Purpose |
| --- | --- | --- |
| Encrypted partition | `/dev/sda1` | Contains the LUKS2 container labelled `ARCH-BACKUP-LUKS` |
| Unlocked mapping | `/dev/mapper/ARCH-BACKUP-LUKS` or `/dev/mapper/luks-...` | Exposes the decrypted ext4 block device |
| Mount point | `/run/media/neon/ARCH-BACKUP` | Exposes the files through the directory tree |

`/dev/dm-4`, if reported by UDisks, is another path to the unlocked block
device. Its number is dynamic and it is not a mount point.

Choose one unlock method. Either unlock the partition through Nautilus, or run:

```bash
udisksctl unlock -b "$backup_partition"
```

Do not run the terminal command if Nautilus already unlocked the partition.
An `Unlocked /dev/sda1 as /dev/dm-4` result confirms that the decrypted block
device exists; it does not identify a directory where files are mounted.
`udisksctl unlock` creates the decrypted block device but does not itself
guarantee that the ext4 filesystem is mounted. Inspect the partition and its
new child directly; the whole-disk `backup_disk` variable is unnecessary for
this check:

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS "$backup_partition"
```

A successful graphical unlock and mount resembles this shortened tree; mapper
names and `/dev/dm-*` numbers may differ:

```text
NAME                  PATH                                  TYPE  FSTYPE      LABEL
sda1                  /dev/sda1                             part  crypto_LUKS ARCH-BACKUP-LUKS
└─ARCH-BACKUP-LUKS    /dev/mapper/ARCH-BACKUP-LUKS         crypt ext4        ARCH-BACKUP
```

The second line must also show `/run/media/neon/ARCH-BACKUP` under
`MOUNTPOINTS` when mounting has completed.

The child with `TYPE=crypt`, `FSTYPE=ext4`, and `LABEL=ARCH-BACKUP` is the
unlocked mapping. UDisks may name it from the LUKS label, such as
`/dev/mapper/ARCH-BACKUP-LUKS`, or from the LUKS UUID, such as
`/dev/mapper/luks-...`; neither name should be assumed. Discover and validate
the exact path:

```bash
backup_mapping="$(
    lsblk -nrpo PATH,TYPE "$backup_partition" |
    awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
printf 'Unlocked mapping: %s\n' "$backup_mapping"
```

If `MOUNTPOINTS` was empty in the preceding `lsblk` output, mount that exact
mapping:

```bash
udisksctl mount -b "$backup_mapping"
```

Nautilus commonly performs both operations as one graphical action, whereas
the terminal flow makes the unlock and mount operations explicit. Discover the
actual mount point instead of inferring it from the mapping name:

```bash
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
test -n "$backup_mount"
printf 'Backup mount point: %s\n' "$backup_mount"
```

For this profile it should print `/run/media/neon/ARCH-BACKUP`. Verify the
source, filesystem, mount options, ownership, and active LUKS mapping before
creating any backup:

```bash
findmnt --target "$backup_mount"
df -hT "$backup_mount"
stat -c '%A %U:%G %n' "$backup_mount"
backup_mapping_name="$(lsblk -dnro NAME "$backup_mapping")"
test -n "$backup_mapping_name"
sudo cryptsetup status "$backup_mapping_name"
```

Required results are an external dm-crypt source, ext4, read-write mount
options and `drwx------ neon:neon`. If the mount point differs, use its exact
path throughout the rest of the chapter.

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
FAT32 or NTFS, or if the underlying external device is not encrypted. Restic
encrypts its own repository and can use many filesystems, but the raw LUKS and
sbctl material created below relies on an encrypted filesystem that preserves
root-only permissions.

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

The initial validated baseline has no separate fallback command-line source.
If a later chapter creates `/etc/kernel/cmdline-fallback`, do not rerun the
commands above against this historical directory. Follow that chapter's
procedure to create a separately named recovery snapshot containing the new
source and the updated boot configuration.

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

If this chapter created the first external disk, its own LUKS2 header also
needs a copy outside that disk. Resolve `backup_partition` again through its
persistent by-id path if this is a new shell, mount the second encrypted device,
and replace the following placeholder with its verified mount point:

```bash
sudo install -d -m 0700 /run/media/neon/SECOND-ENCRYPTED-DISK/backup-disk-recovery
sudo cryptsetup luksHeaderBackup "$backup_partition" --header-backup-file /run/media/neon/SECOND-ENCRYPTED-DISK/backup-disk-recovery/ARCH-BACKUP-LUKS2-header.img
sudo chmod 0600 /run/media/neon/SECOND-ENCRYPTED-DISK/backup-disk-recovery/ARCH-BACKUP-LUKS2-header.img
sudo cryptsetup luksUUID "$backup_partition"
sudo cryptsetup luksUUID /run/media/neon/SECOND-ENCRYPTED-DISK/backup-disk-recovery/ARCH-BACKUP-LUKS2-header.img
```

The two UUIDs must match. Do not store the only copy of this header inside the
LUKS container whose header it is meant to recover. Refresh it after changing
the external disk's keyslots or tokens, and never run `luksHeaderRestore` as a
test.

The second copy does not need to duplicate the full Restic repository on day
one, but truly irreplaceable user data still needs an off-site copy. A later
chapter or handbook guide will compare a second Restic disk, SFTP/NAS, and
supported object-storage backends before automating uploads.

## Close and disconnect the backup disk safely

Finish every backup session by checking that no Restic command or file-manager
operation is still using the disk. Flush pending writes, unmount the ext4
mapping, lock the LUKS partition and finally power off the physical USB disk.
If this is a new shell, repeat the persistent `backup_disk` and
`backup_partition` assignments from **Resolve one persistent device path**.
Validate both before proceeding:

```bash
test -b "$backup_disk"
test -b "$backup_partition"
```

Rediscover the current UDisks mapping instead of assuming that it retained the
name used in an earlier session:

```bash
backup_mapping="$(
    lsblk -nrpo PATH,TYPE "$backup_partition" |
    awk '$2 == "crypt" { print $1; exit }'
)"
test -b "$backup_mapping"
backup_mount="$(findmnt -nr -S "$backup_mapping" -o TARGET)"
test -n "$backup_mount"
```

Then close the complete stack in reverse order:

```bash
sync
udisksctl unmount -b "$backup_mapping"
udisksctl lock -b "$backup_partition"
udisksctl power-off -b "$(readlink -f "$backup_disk")"
```

The unmount and lock operations must succeed before unplugging the cable. If
either reports that the device is busy, identify and close the process instead
of forcing an unmount:

```bash
sudo lsof +f -- "$backup_mount"
```

Keeping the disk disconnected between backup sessions reduces exposure to
accidental deletion, malware and operator error. It does not replace the second
physical copy.

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
- [ ] Any pre-existing NTFS data was copied elsewhere and verified before the dedicated disk was erased.
- [ ] The external disk uses GPT, LUKS2 and ext4 with the reviewed labels.
- [ ] The mounted ext4 root is owned by `neon:neon`, mode `0700`, and is not configured for automatic unlocking or mounting.
- [ ] The external disk's LUKS2 header exists on a second encrypted device and its UUID matches.
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
- [ ] The backup disk can be unmounted, locked and powered off cleanly.

## Sources

- [cryptsetup header backup manual](https://man.archlinux.org/man/cryptsetup-luksHeaderBackup.8.en)
- [cryptsetup LUKS format manual](https://man.archlinux.org/man/cryptsetup-luksFormat.8.en)
- [cryptsetup manual](https://man.archlinux.org/man/cryptsetup.8.en)
- [fdisk manual](https://man.archlinux.org/man/fdisk.8.en)
- [wipefs manual](https://man.archlinux.org/man/wipefs.8.en)
- [mkfs.ext4 manual](https://man.archlinux.org/man/mkfs.ext4.8.en)
- [udisksctl manual](https://man.archlinux.org/man/udisksctl.1.en)
- [Restic documentation](https://restic.readthedocs.io/en/latest/)
- [Restic package](https://archlinux.org/packages/extra/x86_64/restic/)
- [sbctl upstream](https://github.com/Foxboron/sbctl)
- [arch-chroot manual](https://man.archlinux.org/man/arch-chroot.8.en)
