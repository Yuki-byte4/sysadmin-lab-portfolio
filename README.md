# sysadmin-lab-portfolio
# Network and System Management: Lab Portfolio

Hands-on labs for the Network and System Management. Everything was built and tested in virtual machines
(VMware Workstation, Debian 13, Proxmox VE, Windows 11).

## Labs

| Lab | What it covers | Status |
|---|---|---|
| [Lab 1: Virtualization and Docker](lab1-virtualization-docker) | Nested virtualization, Proxmox VE, a Debian VM with MariaDB, Docker Desktop, phpMyAdmin | Done |
| [Lab 2a: Disks, RAID, ZFS and backups](lab2a-disks-raid-zfs-backup) | RAID 6 with hot spares, LVM (linear, mirrored, striped), ZFS, VHDX boot, Back In Time | Done |
| [Lab 2b: Accounts, LDAP, dual boot, systemd](lab2b-accounts-ldap-boot) | Local account lockout, PAM and NSS, OpenLDAP, Windows and Debian dual boot, systemd service | Done |

## Skills shown

- **Virtualization:** VMware Workstation, Proxmox VE, nested virtualization, virtual networks (NAT and host-only)
- **Storage:** mdadm (RAID 6), LVM, ZFS (RAIDZ1, compression), ext4, snapshots and backups
- **Identity:** OpenLDAP, PAM, NSS, local account management
- **Boot and services:** GRUB, dual boot, systemd units, recovery from a locked root account
- **Containers and databases:** Docker, phpMyAdmin, MariaDB

## How each lab is organized

Each lab folder has a README with the goal, the commands used, screenshots as
evidence, and a **Problems and fixes** table describing what went wrong and how
I solved it.

## Notes

- The labs were completed as university coursework. Credentials shown are
  placeholders or values defined by the assignment.
- IP addresses, usernames and device names are generalized on purpose.
- Disk device names (`sdb`, `sdc`, ...) change between reboots, so the commands
  use placeholders and identify disks by size.

## Environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation |
| Linux | Debian 13 (trixie) |
| Windows | Windows 11 |
| Virtualization platform | Proxmox VE 9 |
