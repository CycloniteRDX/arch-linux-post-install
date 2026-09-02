# 03 — Configure storage maintenance and compressed swap

## Goal

Allow periodic TRIM requests from the ext4 filesystems to reach the NVMe SSD,
then add compressed swap in RAM without removing the encrypted disk-backed
swap created by the runbook.

This chapter makes the following changes:

- enables discard propagation through the LUKS mapping;
- keeps continuous ext4 `discard` disabled;
- enables the weekly `fstrim.timer`;
- installs `zram-generator` from the official repositories;
- creates a zstd-compressed zram device sized to half of physical RAM;
- gives zram a higher priority than the existing swap LV;
- disables zswap so it cannot intercept pages before zram;
- rebuilds, signs, and verifies both UKIs;
- reboots once and verifies the complete storage and swap paths.

It does not alter the partition table, LVM layout, ext4 mount options,
swappiness, disk-swap size, or hibernation policy.

## Prerequisites

- Chapters 01 and 02 are complete.
- The normal and fallback UKIs are signed and bootable.
- The Arch installation USB and LUKS passphrase are available.
- `/dev/nvme0n1p2` is the LUKS2 container opened as `cryptlvm`.
- `vg0` contains the `root`, `home`, and `swap` logical volumes.
- The encrypted 16 GiB disk swap is active.

Run the commands as `neon`. Stop if the actual storage names differ from the
canonical profile; do not adapt commands that modify the boot command line
until the difference is understood.

## Resulting policy

| Area | Canonical result |
| --- | --- |
| Filesystem TRIM | Weekly through `fstrim.timer` |
| ext4 continuous discard | Disabled |
| LUKS discard propagation | Enabled at boot |
| zram | Half of RAM, zstd, priority 100 |
| Existing swap LV | Retained as the lower-priority fallback |
| zswap | Explicitly disabled |
| Hibernation | Disabled; no `resume=` parameter |
| VM sysctls | Kernel defaults retained initially |

Discard propagation reveals which encrypted regions are unused, but not the
contents of allocated blocks. This information leak is an accepted trade-off
for SSD maintenance on this personal-laptop profile.

## Audit the starting state

Inspect the active command line, discard path, and swap devices:

```bash
cat /proc/cmdline
sudo cryptsetup status cryptlvm
lsblk -D -o NAME,TYPE,DISC-GRAN,DISC-MAX,DISC-ZERO
swapon --show --output NAME,TYPE,SIZE,USED,PRIO
zramctl
systemctl is-enabled fstrim.timer
```

Before this chapter:

- `/proc/cmdline` has no `rd.luks.options=...=discard`, `zswap.enabled=0`, or
  `resume=` parameter;
- `cryptsetup status` has no `discards` flag;
- the LUKS and LVM rows normally show zero discard granularity and maximum;
- only the encrypted 16 GiB swap LV is active;
- `zramctl` prints no device;
- `fstrim.timer` is disabled.

Non-zero discard values on the physical NVMe rows confirm that the drive
supports the operation. Stop if `nvme0n1` and `nvme0n1p2` both report zero.

## Why periodic TRIM is used

TRIM tells the SSD controller which filesystem blocks no longer contain live
data. This helps the controller manage free flash cells and write
amplification. The intentional unpartitioned tail still provides controller
overprovisioning; TRIM additionally releases deleted blocks inside the
partitioned space.

This profile uses a periodic batch rather than adding `discard` to the ext4
records in `/etc/fstab`. Weekly TRIM avoids issuing a discard for every
filesystem deletion and is sufficient for a normal laptop workload.

Do not change `issue_discards` in `/etc/lvm/lvm.conf`. LVM already passes
filesystem discard requests down to its physical volume. That separate option
controls discards caused by LVM space-management operations and would weaken
the recovery value of `vgcfgrestore`.

## Install zram-generator

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then install the generator through a complete upgrade transaction:

```bash
sudo pacman -Syu zram-generator
```

Read the complete transaction and allow package hooks to finish. Then check
for configuration files that need attention:

```bash
sudo pacdiff --output
pacman -Q zram-generator
```

Resolve any listed `.pacnew`, `.pacsave`, or `.pacorig` files before
continuing.

## Configure zram

Create the local generator configuration:

```bash
sudo micro /etc/systemd/zram-generator.conf
```

Enter exactly:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

Save the file and inspect it:

```bash
cat /etc/systemd/zram-generator.conf
```

The declared size is the maximum uncompressed data capacity of the zram
device. It does not reserve half of RAM at boot. Physical memory is consumed
only as pages are placed in zram, compressed, and accompanied by metadata.

Priority 100 makes the kernel prefer zram to the ordinary low-priority swap
LV. The LV remains available when memory pressure exceeds the useful zram
capacity.

Zstd compression consumes some CPU time, but it can avoid slower NVMe I/O.
The battery effect therefore depends on the workload rather than being an
automatic gain or loss. Start with this conservative configuration and measure
real behavior before changing algorithms or VM tuning.

Do not enable `systemd-zram-setup@zram0.service` manually. It is a generated
static unit and will be created automatically from this configuration at boot.

## Add discard propagation to the UKI command line

Back up the current command line:

```bash
sudo cp -a /etc/kernel/cmdline /etc/kernel/cmdline.before-storage-memory
```

Store the LUKS UUID in the current shell and print it for review:

```bash
luks_uuid=$(sudo cryptsetup luksUUID /dev/nvme0n1p2)
printf '%s\n' "$luks_uuid"
```

Stop if the variable is empty or does not exactly match the UUID already used
by `rd.luks.name=`. Generate the complete canonical line from that variable:

```bash
printf 'rd.luks.name=%s=cryptlvm rd.luks.options=%s=discard zswap.enabled=0 root=/dev/mapper/vg0-root rw\n' "$luks_uuid" "$luks_uuid" | sudo tee /etc/kernel/cmdline
```

This avoids copying a long identifier between screens in a TTY. It writes this
one-line structure, substituting the same UUID in both positions:

```text
rd.luks.name=<LUKS_UUID>=cryptlvm rd.luks.options=<LUKS_UUID>=discard zswap.enabled=0 root=/dev/mapper/vg0-root rw
```

The two UUID values must be identical. The parameters have distinct roles:

| Parameter | Purpose |
| --- | --- |
| `rd.luks.name=...=cryptlvm` | Opens the correct LUKS container with the canonical mapping name. |
| `rd.luks.options=...=discard` | Allows discard requests to pass through that mapping. |
| `zswap.enabled=0` | Prevents zswap from caching pages before they reach zram. |
| `root=/dev/mapper/vg0-root` | Selects the root LV inside the opened container. |
| `rw` | Mounts the real root read-write. |

Do not add `resume=`. The disk swap remains ordinary fallback swap and is not
configured as a hibernation target.

Inspect the result before generating boot artifacts:

```bash
cat /etc/kernel/cmdline
```

Stop if the line contains a placeholder, a wrong UUID, a different mapping or
LV name, a line break, or a `resume=` parameter.

## Rebuild and verify both signed UKIs

Generate the normal and fallback UKIs:

```bash
sudo mkinitcpio -P
```

Both presets must complete successfully, and the sbctl post-hook must sign
both generated EFI files. Warnings about firmware for hardware not present in
the ThinkPad can occur; a failed build or signing operation cannot be ignored.

Verify the embedded command lines:

```bash
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux.efi
sudo bootctl kernel-inspect /boot/EFI/Linux/arch-linux-fallback.efi
```

Both outputs must contain the new discard option, `zswap.enabled=0`, the
correct root LV, and no `resume=` parameter.

Verify the signatures and boot entries:

```bash
sudo sbctl verify
sudo bootctl list
```

The two UKIs and installed systemd-boot executables must be signed. An unsigned
`/boot/vmlinuz-linux` report remains expected because the system boots the
signed UKIs rather than that input file directly.

Do not reboot if either UKI is missing, unsigned, or contains an incorrect
command line.

## Reboot once

After every check succeeds:

```bash
sudo systemctl reboot
```

Enter the LUKS passphrase and log in as `neon`.

## Verify discard propagation after boot

Inspect the active rather than merely configured state:

```bash
cat /proc/cmdline
sudo cryptsetup status cryptlvm
lsblk -D -o NAME,TYPE,DISC-GRAN,DISC-MAX,DISC-ZERO
```

Confirm that:

- `/proc/cmdline` exactly includes the new parameters;
- `cryptsetup status` reports `flags: discards`;
- `cryptlvm` and the three LVM devices now show non-zero discard granularity
  and maximum values;
- the device chain still leads to `/dev/nvme0n1p2`.

The discard flag permits requests to pass through dm-crypt. It does not enable
continuous discard on ext4 and does not itself schedule a TRIM operation.

If `cryptsetup status cryptlvm` reports that the mapping is inactive while the
system is nevertheless running from `vg0-root`, stop the chapter. Zram cannot
rename or close the LUKS mapping. Capture the actual active stack instead of
continuing with an assumed name:

```bash
cat /proc/cmdline
findmnt -no SOURCE /
ls -l /dev/mapper
lsblk -o NAME,PATH,TYPE,FSTYPE,MOUNTPOINTS
sudo dmsetup ls --tree
crypt_name=$(lsblk -rno NAME,TYPE | awk '$2 == "crypt" { print $1; exit }')
printf 'Detected dm-crypt mapping: %s\n' "$crypt_name"
[ -n "$crypt_name" ] && sudo cryptsetup status "$crypt_name"
sudo journalctl -b -u systemd-cryptsetup@cryptlvm.service --no-pager
```

The canonical result is still `crypt_name=cryptlvm`. A different name means
that the embedded command line or the booted UKI does not match the runbook;
an empty value means the block tree needs diagnosis before enabling discard.

## Verify zram and swap priority

Inspect the generated device and service:

```bash
zramctl
cat /sys/block/zram0/comp_algorithm
cat /sys/module/zswap/parameters/enabled
swapon --show --output NAME,TYPE,SIZE,USED,PRIO
free -h
systemctl status systemd-zram-setup@zram0.service --no-pager
```

Confirm that:

- `/dev/zram0` exists and has approximately half of installed RAM as its
  `DISKSIZE`;
- `zstd` appears in brackets as the active compression algorithm;
- zswap reports `N`;
- `/dev/zram0` has priority 100;
- the 16 GiB encrypted disk swap remains active at a lower priority;
- `systemd-zram-setup@zram0.service` is active and exited successfully.

If the zswap parameter file does not exist, the running kernel does not expose
zswap; the required result is still that no zswap cache is active.

`free -h` adds the nominal capacities of both swap devices. That total is not
extra physical RAM, and the zram size is not preallocated memory.

## Run the first filesystem TRIM

Trim each encrypted ext4 filesystem once and display the submitted amount:

```bash
sudo fstrim --verbose /
sudo fstrim --verbose /home
```

Both commands must complete without an unsupported-operation error. The exact
amount varies with filesystem use. The verbose number is the maximum range
submitted through the block stack, not proof that the SSD physically erased
that many bytes at that moment.

Do not run `blkdiscard`; it operates on block devices rather than filesystem
free space and can destroy data when aimed at the wrong target.

Enable the supplied periodic timer:

```bash
sudo systemctl enable --now fstrim.timer
```

Verify its state and next activation:

```bash
systemctl is-enabled fstrim.timer
systemctl is-active fstrim.timer
systemctl list-timers --all fstrim.timer --no-pager
```

The timer must be enabled and active with a future activation. Its supplied
service periodically trims supported mounted filesystems; no custom script or
cron job is required.

## Retain default VM tuning

Record the current values without changing them:

```bash
sysctl vm.swappiness vm.page-cluster
```

Zram-specific tuning such as a high swappiness value or a zero page cluster
can be useful for some workloads, but it is not universally better. This
profile keeps the kernel defaults until memory pressure, latency, power use,
and compression statistics can be measured on the actual laptop.

## Recovery and rollback

If the running system must stop passing discards, restore the saved command
line, rebuild both UKIs, verify their signatures, and reboot:

```bash
sudo cp -a /etc/kernel/cmdline.before-storage-memory /etc/kernel/cmdline
sudo mkinitcpio -P
sudo sbctl verify
sudo systemctl reboot
```

If a command-line mistake prevents both UKIs from booting, use the verified
Arch installation USB and the runbook's mount-and-chroot recovery procedure.
Restore the same backup from inside the chroot and run `mkinitcpio -P` there.

To stop using zram while retaining the package and disk swap, stop the device,
disable its local configuration by renaming it, and reload systemd:

```bash
sudo systemctl stop systemd-zram-setup@zram0.service
sudo mv /etc/systemd/zram-generator.conf /etc/systemd/zram-generator.conf.disabled
sudo systemctl daemon-reload
swapon --show
```

The encrypted disk swap must remain active before continuing normal use.

## Completion checkpoint

```bash
cat /proc/cmdline
sudo cryptsetup status cryptlvm
lsblk -D -o NAME,TYPE,DISC-GRAN,DISC-MAX,DISC-ZERO
zramctl
swapon --show --output NAME,TYPE,SIZE,USED,PRIO
systemctl is-enabled fstrim.timer
systemctl is-active fstrim.timer
systemctl --failed
sudo sbctl verify
```

The chapter is complete when:

- discard support reaches every layer from ext4 through LVM and LUKS to NVMe;
- the ext4 mount records still omit continuous `discard`;
- `fstrim.timer` is enabled and active;
- zram uses zstd, half of RAM, and priority 100;
- zswap is disabled;
- the encrypted disk swap remains active at a lower priority;
- no hibernation or resume parameter was introduced;
- both UKIs contain the intended command line and remain signed;
- no system unit is failed.

## Sources

- [ArchWiki: Solid state drive](https://wiki.archlinux.org/title/Solid_state_drive)
- [ArchWiki: zram](https://wiki.archlinux.org/title/Zram)
- [ArchWiki: Swap](https://wiki.archlinux.org/title/Swap)
- [ArchWiki: dm-crypt system configuration](https://wiki.archlinux.org/title/Dm-crypt/System_configuration)
- [Arch manual: systemd-cryptsetup-generator(8)](https://man.archlinux.org/man/systemd-cryptsetup-generator.8)
- [Arch manual: crypttab(5)](https://man.archlinux.org/man/crypttab.5)
- [Arch manual: zram-generator.conf(5)](https://man.archlinux.org/man/zram-generator.conf.5)
- [Arch manual: fstrim(8)](https://man.archlinux.org/man/fstrim.8)
- [Arch package: zram-generator](https://archlinux.org/packages/extra/x86_64/zram-generator/)

## Next step

Continue with chapter 04 to configure firewalld, inspect listening sockets,
and define the initial network-hardening boundary without enabling remote SSH
access.
