# 01 — Audit the runbook handoff

## Goal

Confirm that the system received from `arch-linux-runbook` matches the
post-installation profile before changing packages, boot configuration,
storage, networking, or services.

This chapter is an inspection checkpoint. It does not install packages, edit
configuration files, enable services, or rebuild boot artifacts.

> [!IMPORTANT]
> The expected results describe a fresh handoff from the current runbook. An
> older or already customized Arch installation may legitimately contain
> discard propagation, zram, Niri, greetd, or other later configuration. Record
> that machine as a migration case; do not remove working configuration merely
> to make this audit resemble a fresh installation.

## Prerequisites

- The runbook reached its successful first-boot checkpoint.
- The machine is booted from the normal `arch-linux.efi` UKI.
- Secure Boot is enabled.
- The regular user is logged in at a TTY.
- Networking can be established with NetworkManager.

Run the commands as the regular user `neon`. Use `sudo` only where shown. Stop
at the first unexplained difference and reconcile it with the runbook before
continuing.

## Confirm the user and administrative access

```bash
whoami
id
sudo -k
sudo -v
```

Confirm that:

- `whoami` prints `neon`;
- `id` includes the `wheel` group;
- `sudo -v` requests `neon`'s password and succeeds.

Do not continue from a persistent root shell. The remaining procedure assumes
ordinary user sessions with explicit privilege escalation.

## Confirm the machine identity and localization

```bash
hostnamectl
localectl
timedatectl
uname -r
```

Confirm that:

- the hardware is the intended ThinkPad;
- each laptop has its intended unique hostname;
- the system locale is `en_US.UTF-8`;
- the console keymap is `us` or `es`, matching the physical keyboard;
- the time zone is `Europe/Madrid`;
- the kernel is the installed Arch `linux` kernel;
- the clock is credible and the NTP service is active.

The clock may need a short time after networking becomes available before
`timedatectl` reports `System clock synchronized: yes`.

## Confirm networking and time synchronization

```bash
nmcli device status
nmcli connection show --active
ping -c 3 archlinux.org
timedatectl
```

At least one intended network connection must be active. The ping must resolve
the hostname and receive replies, and `timedatectl` must eventually report an
active NTP service and a synchronized clock.

If networking does not work, stop here and repair the NetworkManager connection
using the runbook's first-boot chapter. Do not install another network manager
as a workaround.

## Confirm the boot and trust chain

Because the EFI System Partition is mounted with root-only permissions, use
`sudo` for every command that needs to inspect files below `/boot`:

```bash
sudo bootctl status
sudo bootctl list
sudo bootctl --print-loader-path
sudo sbctl status
sudo sbctl verify
```

Confirm that:

- Secure Boot is enabled and Setup Mode is disabled;
- the enrolled vendor certificates include Microsoft;
- the current and default entry is `arch-linux.efi`;
- `arch-linux-fallback.efi` is also listed;
- the loader path is `/boot/EFI/systemd/systemd-bootx64.efi`;
- both UKIs and every installed systemd-boot executable are signed.

`sbctl verify` may report `/boot/vmlinuz-linux` as unsigned. This is expected:
the file is an input used to build the signed UKIs, and no boot entry executes
it directly.

Inspect the active kernel command line:

```bash
cat /proc/cmdline
```

It must contain all of the following elements, with the actual LUKS UUID in
place of the placeholder:

```text
rd.luks.name=<LUKS_UUID>=cryptlvm
root=/dev/mapper/vg0-root
rw
```

At this stage it must not contain `resume=` or a LUKS `discard` option.
Discard propagation is introduced deliberately in chapter 03; hibernation is
outside the canonical profile.

## Confirm the storage stack

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo cryptsetup status cryptlvm
findmnt /
findmnt /home
findmnt /boot
swapon --show
```

Confirm the complete chain:

| Layer | Expected result |
| --- | --- |
| Disk | `/dev/nvme0n1`, approximately 476.9 GiB |
| EFI System Partition | `/dev/nvme0n1p1`, 1 GiB, vfat, mounted at `/boot` |
| Encrypted partition | `/dev/nvme0n1p2`, 400 GiB, LUKS2 |
| Open mapping | `/dev/mapper/cryptlvm` backed by `/dev/nvme0n1p2` |
| Root | `/dev/mapper/vg0-root`, ext4, mounted at `/` |
| Home | `/dev/mapper/vg0-home`, ext4, mounted at `/home` |
| Disk swap | The 16 GiB `vg0/swap` logical volume is active |

`swapon --show` may display a `/dev/dm-*` path rather than `/dev/vg0/swap`.
Use the `lsblk` tree to identify its parentage. No zram device is expected yet.

Inspect the LVM allocation:

```bash
sudo pvs -o pv_name,vg_name,pv_size,pv_free
sudo vgs -o vg_name,vg_size,vg_free
sudo lvs -o lv_name,vg_name,lv_size,lv_attr
```

Confirm that:

- `/dev/mapper/cryptlvm` is the only physical volume in `vg0`;
- `vg0` contains exactly `root`, `home`, and `swap`;
- root is 192 GiB and swap is 16 GiB;
- home occupies the remaining planned space;
- approximately 256 MiB remains free in `vg0`.

The approximately 75.9 GiB unpartitioned SSD tail is also intentional. Do not
extend partition 2 or any logical volume merely because free space is visible.

## Confirm mounts, capacity, and fstab

```bash
df -h / /home /boot
findmnt -no SOURCE,FSTYPE,OPTIONS /
findmnt -no SOURCE,FSTYPE,OPTIONS /home
findmnt -no SOURCE,FSTYPE,OPTIONS /boot
cat /etc/fstab
```

Confirm that root and home are writable ext4 filesystems and that `/boot` is a
writable vfat filesystem. The `/boot` options must include both
`fmask=0177` and `dmask=0077`.

The ext4 records must not contain the continuous `discard` mount option. The
swap record must refer to the intended encrypted logical volume by UUID. Stop
if `/`, `/home`, or `/boot` is unexpectedly close to full.

Verify the effective ESP permissions:

```bash
sudo stat -c '%a %U:%G %n' /boot /boot/loader /boot/loader/random-seed
```

The directories should appear as mode `700` and the random-seed file as mode
`600`, all owned by `root:root`.

## Confirm the base packages

```bash
pacman -Q base linux linux-firmware amd-ucode
pacman -Q mkinitcpio systemd systemd-ukify sbctl
pacman -Q cryptsetup lvm2 e2fsprogs dosfstools efibootmgr
pacman -Q networkmanager sudo micro man-db man-pages
```

Every package must be present. Versions are recorded for diagnosis but are not
pinned by this project. Do not update only one listed package; the complete
system upgrade belongs to chapter 02.

## Confirm the base services

```bash
systemctl is-enabled NetworkManager.service systemd-timesyncd.service systemd-boot-update.service
systemctl is-active NetworkManager.service systemd-timesyncd.service
systemctl --failed
```

The three services must be enabled. NetworkManager and systemd-timesyncd must
be active, and `systemctl --failed` must report no failed system units.

`systemd-boot-update.service` is a boot-time one-shot service. It does not need
to remain active after completing successfully.

## Completion checkpoint

Continue only when all of these statements are true:

- The intended user has password-protected `sudo` access.
- Hostname, English locale, physical-keyboard keymap, time zone, and clock are
  correct.
- NetworkManager provides working DNS and internet connectivity.
- The normal signed UKI and intended signed systemd-boot path are active under
  Secure Boot.
- The fallback UKI is available and signed.
- LUKS2, LVM, ext4, the ESP, and disk swap match the canonical layout.
- The ESP has root-only permission masks.
- The kernel command line contains neither hibernation nor discard options.
- The expected base packages are installed.
- The three base services are enabled and no system units are failed.
- A verified Arch installation USB and the LUKS passphrase remain available
  for recovery.

This checkpoint creates no new backup. LUKS-header, sbctl-key, and user-data
backups are handled in chapter 12 after their destinations and protection have
been defined.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: Installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch manual: bootctl(1)](https://man.archlinux.org/man/bootctl.1)
- [Arch manual: sbctl(8)](https://man.archlinux.org/man/sbctl.8)
- [Arch manual: cryptsetup-status(8)](https://man.archlinux.org/man/cryptsetup-status.8)
- [Arch manual: lsblk(8)](https://man.archlinux.org/man/lsblk.8)
- [Arch manual: findmnt(8)](https://man.archlinux.org/man/findmnt.8)
- [Arch manual: systemctl(1)](https://man.archlinux.org/man/systemctl.1)

## Next step

Continue with chapter 02 to perform the first complete system upgrade and
establish the package-maintenance, Arch news, mirror, package-cache, Git, SSH
client, documentation, and AUR policies.
