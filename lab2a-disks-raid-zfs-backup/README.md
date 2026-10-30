# Lab 2a: Disks, RAID, LVM, ZFS and Backups

**Environment:** Debian 13 (trixie) and Windows 11 VMs in VMware Workstation

## Contents
1. [Theory: ZFS, RAID, hot spare](#1-theory)
2. [Windows VHDX dual instance](#2-windows-inside-windows-vhdx)
3. [RAID 6 and LVM](#3-raid-6-and-lvm)
4. [Physical extents](#4-physical-extents-per-physical-volume)
5. [ZFS](#5-zfs)
6. [Local backups](#6-local-backups)
7. [Problems and fixes](#7-problems-and-fixes)

> **Note on device names.** Disk letters (`sdb`, `sdc`, ...) changed between
> reboots and after adding disks. Commands below use `DISKn` placeholders.
> Always identify disks by size with `lsblk` before running destructive commands.

---

## 1. Theory

### ZFS
ZFS (Zettabyte File System) is a file system with built-in volume management.
Its code was open-sourced in 2005 after Solaris and later ported to Linux,
macOS and FreeBSD. Storage management has two sides: managing the physical
volumes, and managing the files stored on logical block devices. ZFS unifies
both, acting as volume manager and file system, so it knows about the physical
disks and every file stored on them. It also provides dataset- and pool-level
snapshots, replication and snapshot cloning, one of its most powerful tools.

### RAID
RAID (Redundant Array of Independent Disks) coordinates two or more drives into
one array instead of storing everything on a single disk. Its main purpose is
protection against disk failure by storing extra recovery data across the
disks. For example, RAID 6 survives two simultaneous disk failures without
data loss.

### Hot spare
A hot spare is an extra disk kept ready in the array. When an active disk fails,
it automatically takes over and the array rebuilds, shortening the time spent
in a degraded state. It is like a UPS: it steps in when the main supply fails.
It makes sense in every level with redundancy (RAID 1, 5, 6, 10) and not in
RAID 0, which has nothing to rebuild.

---

## 2. Windows inside Windows (VHDX)

The task: create a 16 GB VHDX inside the Windows VM, install a second Windows
instance on it, and boot either one from the boot menu.

1. **Disk Management → Action → Create VHD.** Name it, choose **VHDX**,
   create it, initialize the disk and create an NTFS volume (letter `V:`).
2. Mount the Windows ISO in the VM's CD/DVD drive and apply the image to `V:`
   from an administrator Command Prompt:

```
dism /apply-image /imagefile:D:\sources\install.wim /index:1 /applydir:V:\
bcdboot V:\Windows
bcdedit /enum
```

3. Reboot and pick either instance from the boot menu.

![Create VHD](screenshots/create-vhd.png)
![dism apply-image](screenshots/dism.png)
![Boot menu with both Windows instances](screenshots/windows-boot-menu.png)

---

## 3. RAID 6 and LVM

### Setup
A Debian VM with a system disk plus six extra 100 MB disks. `lsblk` shows them
before and after adding the disks (six new icons also appear in the VMware
status bar).

![lsblk before](screenshots/lsblk-before.png)
![lsblk after](screenshots/lsblk-after.png)

### (a) Create the RAID 6 array
`mdadm` manages software RAID. The array has 4 active disks and 2 spares, and
is saved to the configuration so it keeps the name `md0` after a reboot.

```bash
apt install mdadm lvm2
mdadm --create /dev/md0 --level=6 --raid-devices=4 --spare-devices=2 DISK1 DISK2 DISK3 DISK4 DISK5 DISK6
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

### (b) Check the layout
```bash
cat /proc/mdstat
mdadm --detail /dev/md0
```
Expected: `Raid Level: raid6`, `Raid Devices: 4`, `Spare Devices: 2`.

![mdadm detail](screenshots/mdadm-detail.png)

### (c)(d) Two 200 MB disks and physical volumes
Two 200 MB disks were added in VMware and a rescan was triggered. Physical
volumes were created on `/dev/md0` and on both disks.

```bash
echo "- - -" | tee /sys/class/scsi_host/host*/scan
pvcreate /dev/md0 EXTRA1 EXTRA2
```

### (e) Volume group
```bash
vgcreate vg0 /dev/md0 EXTRA1
```

### (f) Linear volume lv0
```bash
lvcreate -n lv0 -L 200M vg0
vgdisplay vg0
```
![vgdisplay](screenshots/vgdisplay.png)

### (g) ext4 and mount at boot
ext4 is the Linux file system that organizes files into folders and tracks where
the data lives on disk. lv0 is formatted and mounted on `/mnt/vg0-lv0`. The
`nofail` option stops a missing volume from blocking boot.

```bash
mkfs.ext4 /dev/vg0/lv0
mkdir -p /mnt/vg0-lv0
echo '/dev/vg0/lv0 /mnt/vg0-lv0 ext4 defaults,nofail 0 2' >> /etc/fstab
mount -a
df -h /mnt/vg0-lv0
```
I forgot to screenshot these commands when I first ran them, so the screenshot
below is a later proof that the file system exists.

![ext4 proof](screenshots/ext4-proof.png)

### (h) Add the second 200 MB disk
```bash
vgextend vg0 EXTRA2
vgs
```
The volume group size increased.

### (i) Mirrored volume lv1
A mirrored volume with 1 mirror and 52 MB. The free space of `vg0` must drop by
exactly 2 × 52 MB. `--mirrorlog core` keeps the log in memory so the check is
exact. With the default `raid1` type, LVM also uses a few extents for metadata
and the numbers would not match.

```bash
vgs                      # note the free space
lvcreate -L 52M -m1 --type mirror --mirrorlog core -n lv1 vg0
vgs                      # free space is 104 MB lower
```

### (j) Striped volume lv2
Mirroring and striping distribute data over several disks in different ways.
Mirroring is safer (the same data on two disks); striping is faster (data is
split across disks).

```bash
lvcreate -L 190M -i 2 -n lv2 vg0
```
LVM rounds 190 MB up to 192 MB, a multiple of the 4 MB extent size.

### (k) Grow lv0 by 60 MB
The `-r` option resizes the file system in the same step. The manual way is
`lvextend -L +60M` followed by `resize2fs /dev/vg0/lv0`.

```bash
lvextend -L +60M -r /dev/vg0/lv0
df -h /mnt/vg0-lv0
```

### (l) Fail a disk
One active disk is marked as failed to show that a spare takes its place.

```bash
mdadm /dev/md0 --fail DISK1
mdadm --detail /dev/md0
mdadm /dev/md0 --remove DISK1
mdadm --detail /dev/md0
```
Expected: the failed disk is removed, a former spare is now active, and one
spare remains.

![Failure and spare rebuild](screenshots/mdadm-fail.png)

---

## 4. Physical extents per physical volume

Think of LVM as a warehouse: physical volumes are the floors, physical extents
are the boxes, and logical volumes are the storage areas. To see which extents
are used and by which logical volume:

```bash
pvdisplay -m
pvs --segments -o pv_name,lv_name,seg_start_pe,pvseg_size,segtype
pvs -o pv_name,pv_pe_count,pv_pe_alloc_count
```
`pvdisplay -m` maps physical extents to logical volumes. The second command lists
each segment by volume, start and size. The third shows the number of used
extents per physical volume.

![pvdisplay -m](screenshots/pvdisplay.png)

---

## 5. ZFS

I first worked on the same VM, then went back to correct a mistake and created a
**clone**. Five 1 GB virtual disks were added to the clone with the **Add**
button in the VM settings, in addition to the system disk.

### (a) Install OpenZFS
First check the Debian version. On versions below 13 the `contrib` and
`non-free` components must be added to the sources list. My install had a
sources list without `contrib`, so apt reported "Unable to locate package" until
I added it. The headers are installed first because DKMS builds the ZFS module
against them, which takes a few minutes.

```bash
sed -i 's/ main / main contrib non-free /' /etc/apt/sources.list
apt update
apt install linux-headers-amd64
apt install zfs-dkms zfsutils-linux
zfs version
```

### (b) Create the pool
A pool named `userpool` in RAIDZ1 across 3 disks, with a mirrored log on the
fourth and fifth disks.

```bash
zpool create userpool raidz1 DISK1 DISK2 DISK3 log mirror DISK4 DISK5
zpool status userpool
```
![zpool status](screenshots/zpool-status.png)

### (c) Compressed dataset
```bash
zfs create -o compression=lz4 userpool/users
zfs get compression userpool/users
```

### (d) Confirm the mounts
```bash
zfs list
df -h | grep userpool
```
Both `/userpool` and `/userpool/users` must appear. If not, repeat the previous
step or reboot.

![zfs list and df](screenshots/zfs-mounts.png)

### (e) I/O benchmark
I/O benchmarks measure how fast a storage device reads and writes data. A 10 MB
file of random data was written with `dd` on both setups. `conv=fdatasync` makes
`dd` wait until the data is flushed to disk, so the speed is realistic.

```bash
dd if=/dev/urandom of=/tmp/random10M bs=1M count=10
dd if=/tmp/random10M of=/userpool/users/test bs=1M conv=fdatasync
dd if=/tmp/random10M of=/mnt/vg0-lv0/test bs=1M conv=fdatasync
```

| Setup | Write speed |
|---|---|
| LVM on RAID 6 | `<fill in>` |
| ZFS RAIDZ1 + mirrored log | `<fill in>` |

Random data does not compress, so lz4 gives no advantage here. With only 10 MB
the results are noisy, so I repeated each test several times. The two setups
were also on different VMs, so the comparison is only indicative.

![dd results](screenshots/dd-benchmark.png)

---

## 6. Local backups

### Deduplication
Have you ever used the phone gallery feature that finds duplicate pictures so
you can delete them? Deduplication is the same idea for storage: a technique
that detects duplicate copies of data and keeps only one copy, saving space.
It can work on whole files or on blocks, and happens inline (while writing) or
afterwards. It saves a lot of space for backups and VM images, at the cost of
extra CPU and memory.

### A time machine for Debian
Time Machine is the macOS backup tool. On Debian I used **Back In Time**, which
does the same job: it backs up files to external or network storage and lets you
recover lost files or older versions. My three main features are automatic
backups, easy file recovery and restoring older states.

Since the VM has no desktop, I used the command-line version with an extra disk
mounted on `/mnt/backup`:

```bash
apt install backintime-common
backintime backup
backintime snapshots-list
```

I then changed one file and deleted another, took a second snapshot, compared
the two and restored the deleted file. Back In Time stores snapshots as normal
folders and uses hard links, so an unchanged file has the **same inode number**
in every snapshot. It is stored once, which is deduplication at file level:

```bash
find /mnt/backup -name file3.txt -exec ls -li {} \; 2>/dev/null
```

![Snapshot list](screenshots/snapshots-list.png)
![Restored file](screenshots/restored-file.png)
![Same inode in both snapshots](screenshots/inode.png)

---

## 7. Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| `mdadm: not enough devices` | The six disks were not attached yet | Add the disks, rescan, check `lsblk` |
| `pvcreate` refused two disks ("md component") | Device letters had shifted | Identify disks by size with `lsblk` |
| `/dev/md0` missing after reboot | Array not assembled or not saved | `mdadm --assemble --scan`, save to `mdadm.conf`, `update-initramfs -u` |
| Emergency mode at boot | `fstab` entry for a volume that did not exist | Add `nofail`, fix the array |
| `Unable to locate package zfs-dkms` | `contrib` missing from sources | Add `contrib non-free`, run `apt update` |
| Backup config crashed with `KeyError` | Trailing space in the config file | Strip it with `sed -i 's/[[:space:]]*$//'` |
| `mkfs` refused: "in use by the system" | Wrong disk (the system disk) | Check `lsblk` first, format only an empty disk |

## What I learned
- Identify disks by size and `lsblk`, never by letter alone.
- Save RAID configuration so the array survives reboots.
- Hard links explain how snapshot tools and deduplication save space.
# Back In Time Demonstration

Part of [Lab 2a](README.md). Back In Time is a snapshot backup tool for Linux.
The VM has no desktop, so I used the command-line version (`backintime-common`,
version 1.5.5) with an extra disk mounted on `/mnt/backup` as the destination.

## Steps

1. Create a test folder `~/Documents` with a few files (`file1`, `file2`, `file3`).
2. Take a first snapshot.
3. Edit `file1`, delete `file2` and take a second snapshot.
4. Browse the snapshots, compare the versions and restore the deleted file.

## 1. Test files

    mkdir -p ~/Documents
    echo "version1" > ~/Documents/file1.txt
    echo "delete"   > ~/Documents/file2.txt
    echo "keep"     > ~/Documents/file3.txt

I first typed `exho` instead of `echo`, which gave `command not found`, and
repeated the command correctly.

![Creating the test files](screenshots/bit-test-files.png)

## 2. Configuration

Back In Time reads its settings from `~/.config/backintime/config`. The folder
did not exist, so I created it first, then wrote a minimal config with the
destination (`/mnt/backup`) and the folder to back up (`~/Documents`):

    mkdir -p ~/.config/backintime
    cat > ~/.config/backintime/config << 'EOF'
    config.version=6
    profiles.version=1
    profile1.snapshots.mode=local
    profile1.snapshots.path=/mnt/backup
    profile1.snapshots.include.size=1
    profile1.snapshots.include.1.type=0
    profile1.snapshots.include.1.value=/home/<USER>/Documents
    profile1.snapshots.exclude.size=0
    profile1.snapshots.automatic_backup_mode=0
    EOF

![Configuration file](screenshots/bit-config.png)

Then I validated it:

    backintime check-config

It prepared the snapshot path, checked the config and installed the crontab,
and reported that the main profile is fine.

![check-config output](screenshots/bit-check-config.png)

## 3. First snapshot

    backintime backup
    backintime snapshots-list

The warning "Inhibit Suspend failed" is harmless: there is no desktop session
on this VM to block sleep.

![First snapshot](screenshots/bit-first-snapshot.png)

## 4. Edit one file, delete another, second snapshot

    echo "version2" > ~/Documents/file1.txt
    rm ~/Documents/file2.txt
    backintime backup
    backintime snapshots-list

I mistyped the command once as `snapshot-list`, and Back In Time listed the
valid commands (the correct one is `snapshots-list`). Now two snapshot IDs
appear. If nothing has changed, Back In Time reports "Nothing changed, no new
snapshot necessary" and creates no snapshot.

![Second snapshot and the list of two snapshots](screenshots/bit-second-snapshot.png)

## 5. Compare the versions

Snapshots are normal folders, so `find` shows where each file is stored:

    find /mnt/backup -name file1.txt 2>/dev/null
    find /mnt/backup -name file2.txt 2>/dev/null

`file1.txt` appears in **both** snapshots (with different content), while
`file2.txt` appears **only in the first** one, because I deleted it before the
second snapshot. The `lost+found: Permission denied` message is normal: that
folder belongs to root.

![find results](screenshots/bit-find.png)

## 6. Restore the deleted file

    cp /mnt/backup/backintime/<HOST>/<USER>/1/<FIRST_SNAPSHOT_ID>/backup/home/<USER>/Documents/file2.txt ~/Documents/
    cat ~/Documents/file2.txt

![Restored file](screenshots/bit-restore.png)

## 7. Hard links (link to deduplication)

`file3.txt` did not change between snapshots. Listing it in both snapshots
shows the **same inode number**, so the data is stored only once:

    find /mnt/backup -name file3.txt -exec ls -li {} \; 2>/dev/null

![Same inode in both snapshots](screenshots/bit-inode.png)

## Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| `echo: command not found` | Typo (`exho`) | Retype the command |
| `No such file or directory` writing the config | `~/.config/backintime` did not exist | `mkdir -p ~/.config/backintime` |
| `KeyError: 'local '` | Trailing space pasted into the config | `sed -i 's/[[:space:]]*$//' ~/.config/backintime/config` |
| "No snapshots in main profile" | No backup had been run yet | Run `backintime backup` first |
| `invalid choice: 'snapshot-list'` | Wrong command name | Use `snapshots-list` |
| "Nothing changed, no new snapshot" | Files unchanged, or edited as the wrong user | Run as the same user and change a file in the backed-up folder |
