# Lab 2: Accounts, LDAP, Dual Boot and systemd

## Local Accounts and File-system Security

### 1(a) Disabling Administrator in Windows
Administrator can be disabled with:

    Disable-LocalUser -Name "Administrator"

This is not recommended if you have only one user account on the PC, because you
could lose the administrative privileges needed to manage the system and
troubleshoot problems.

### 1(b) Disabling root in Linux
In Windows there is an Administrator; in Linux the superuser is called root. To
deactivate it:

    sudo passwd -l root
    sudo usermod -L root

`-l` locks the password by putting `!` in front of the hash. I don't recommend
this, because you lose privileges such as installing, updating or deleting
packages and files, and changing the system.

**To get root back**, open the Debian boot menu (GRUB) (corrected: GRUB, not
BIOS). As soon as the menu appears press `e` to edit. Find the line starting
with `linux /boot/vmlinuz...  root=UUID=... ro quiet` and replace `ro quiet`
with `rw init=/bin/bash` so the disk is writable. Boot with Ctrl+X, then run:

    mount -o remount,rw /
    passwd -u root
    usermod -s /bin/bash root
    passwd root

`passwd root` asks for a new password. Then `sync` and reboot.

### 1(c) Can root still log in?
(corrected) Root cannot log in **with its password**, because the password hash
was made unusable and trying gives an authentication failure. The account
itself still exists and is not fully blocked: it can still be reached with
`sudo -i` or `su` from an admin user, or with SSH keys. Root privileges are
sensitive, so be careful when locking them.

## PAM and NSS (question 2)

- **PAM (Pluggable Authentication Modules)** checks how users are authenticated.
  It uses separate modules to verify the password and apply security rules
  without the user noticing.
- **NSS (Name Service Switch)** controls where to look for user and group
  information such as UID, GID, usernames and group memberships.

Local accounts are stored in `/etc/passwd` with password hashes in
`/etc/shadow`. Local groups are in `/etc/group`. NSS uses a configuration file
to decide where to find account and group information, often local files. PAM
also uses configuration files, but to decide how authentication is performed,
including checking passwords and applying login rules.

## OpenLDAP on Debian

### Install slapd
    sudo apt install slapd

It asks for an administrator password, which I set to `asd123`.

![slapd installation](screenshots/slapd-install.png)
![slapd password prompt](screenshots/slapd-password.png)

### Install curl and run the README script
    apt install curl
    /bin/bash -c "$(curl -fsSL https://gitlab.com/ipb-gsr/helper-scripts/-/raw/main/slapd-init/prepare.sh)"

(My first attempt had a typo in the URL and returned error 403. The corrected
command worked: it stopped the LDAP server, fetched `config.ldif` and
`data.ldif`, and started the server again.)

![curl install and script output](screenshots/curl-script.png)

### Check existing users
`ldapsearch` shows the two existing students with their UIDs and group:

    ldapsearch -LLL -x -b 'ou=users,dc=ipb,dc=pt' dn uidNumber memberUid

![ldapsearch output](screenshots/ldapsearch.png)

### Add student03
I created `student03.ldif` with nano and added it with `ldapadd`, then set the
password with `ldappasswd`:

    ldapadd -x -H ldap://127.0.0.1 -D "cn=manager,dc=ipb,dc=pt" -w asd123 -f student03.ldif
    ldappasswd -x -H ldap://127.0.0.1 -D "cn=manager,dc=ipb,dc=pt" -W -S "<DN of student03>"

The LDIF file is in [`ldif/student03.ldif`](ldif/student03.ldif).

![student03.ldif](screenshots/student03-ldif.png)
![ldapadd and ldappasswd](screenshots/ldapadd.png)

### libpam-ldap and libnss-ldap
To let users authenticate against the LDAP server while keeping local users
working, I installed the LDAP PAM and NSS modules (Debian selects
`libpam-ldapd` and `libnss-ldapd` instead):

    apt install libpam-ldap libnss-ldap

In the configuration screens I set the LDAP server URI to `ldap://127.0.0.1/`
and selected `passwd`, `group` and `shadow` for the NSS lookups.

![nslcd configuration](screenshots/nslcd-config.png)
![libnss-ldapd configuration](screenshots/libnss-config.png)

### Home directory at first login
To create the home directory on login I used Debian's PAM tool and ticked
**Create home directory on login** (plus Unix and LDAP authentication):

    pam-auth-update

![pam-auth-update](screenshots/pam-auth-update.png)

### Verification
`getent`, `id` and `ldapsearch` show that student03 exists and is visible to the
system, in the group `students`:

    getent passwd student03
    id student03
    getent group 2000
    ldapsearch -x -LLL -H ldap://127.0.0.1 -b dc=ipb,dc=pt dn

Logging in as the three students created their home directories:

    su - student03
    su - student01
    su - student02

![getent and id](screenshots/getent-id.png)
![ldapsearch all entries](screenshots/ldapsearch-all.png)
![su logins](screenshots/su-logins.png)

## Boot and Shutdown: dual boot

The task asks for a VM with enough disk space for Windows and Debian to coexist,
installing Linux first, then Windows, with a boot menu that belongs to Linux and
also offers Windows. I created a VM with an empty ISO section and **60 GB**
(SATA disk type), then booted the Debian ISO.

In the installer I followed the defaults until **Partition disks**. There I
created a new partition table and one **20 GB ext4 partition mounted on `/`**
(bootable), leaving the rest as free space for Windows.

![VM hardware](screenshots/dualboot-hardware.png)
![Debian installer partitioning](screenshots/partition-disks.png)
![Partition overview](screenshots/partition-overview.png)

Then I changed the VM's ISO to the Windows one and booted again. Right after the
VMware logo, press **Esc** for the boot menu, select the CD-ROM drive, select
install and continue.

![Boot menu](screenshots/boot-menu.png)

In Windows setup I chose **Custom: Install Windows only (advanced)**, selected
**Unallocated Space** and clicked Next, without touching the Debian partition.

I got an error saying the PC must support TPM 2.0 and Secure Boot. I bypassed it
from the setup command prompt: `regedit` → `HKEY_LOCAL_MACHINE\SYSTEM\Setup` →
create a key `LabConfig` with the DWORDs `BypassTPMCheck`,
`BypassSecureBootCheck`, `BypassCPUCheck` and `BypassRAMCheck`, all set to `1`.

![regedit in setup](screenshots/regedit.png)
![LabConfig values](screenshots/labconfig.png)

I made a mistake by giving the disk less space than needed, so I expanded the
disk from the VM hardware settings to 90 GB, and then it ran without problems.

![Select location](screenshots/select-location.png)
![Installing Windows 11](screenshots/installing-windows.png)

**Result:** the boot menu that appears belongs to Linux (GRUB) and it also shows
**Windows Boot Manager**. I recommend taking VM snapshots in case something
crashes.

![GRUB menu with Windows](screenshots/grub-menu.png)
![VM snapshots](screenshots/snapshots.png)

## systemd service (`/usr/bin/xpto`)

A service is described by a unit file. I created it with nano at
`/etc/systemd/system/xpto.service` (corrected path). It has three parts:
**Unit** (description and ordering), **Service** (how to run it; `Type=simple`
is for a foreground process) and **Install** (which target pulls it in when it
is enabled). The file is in [`systemd/xpto.service`](systemd/xpto.service).

Then I enabled and started it:

    sudo systemctl daemon-reload
    sudo systemctl enable --now xpto.service
    systemctl status xpto.service
    systemctl is-enabled xpto

![xpto.service in nano](screenshots/xpto-nano.png)
![enable and status](screenshots/xpto-status.png)
