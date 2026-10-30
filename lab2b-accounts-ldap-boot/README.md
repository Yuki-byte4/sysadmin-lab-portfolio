# Lab 2b: Accounts, LDAP, Dual Boot and systemd

**Environment:** Debian 13 (trixie) and Windows 11 VMs in VMware Workstation

## Contents
1. [Local accounts and file-system security](#1-local-accounts-and-file-system-security)
2. [PAM and NSS](#2-pam-and-nss)
3. [OpenLDAP with PAM and NSS](#3-openldap-with-pam-and-nss)
4. [Dual boot: Debian and Windows](#4-dual-boot-debian-and-windows)
5. [systemd service](#5-systemd-service)
6. [Problems and fixes](#6-problems-and-fixes)

---

## 1. Local accounts and file-system security

### (a) Disable Administrator in Windows
From an administrator command line:

    Disable-LocalUser -Name "Administrator"

I don't recommend doing this when you have only one administrator account,
because you could lose the privileges needed to manage the system and
troubleshoot problems. Create another administrator first.

### (b) Disable root in Debian
In Windows the privileged account is called Administrator; in Linux it is
`root`. To lock it:

    sudo passwd -l root
    sudo usermod -L root

The `-l` option locks the password by putting a `!` in front of the hash in
`/etc/shadow`. Optionally the shell can be removed too:
`usermod -s /usr/sbin/nologin root`. Check the result with `passwd -S root`
(status `L` means locked).

I don't recommend this on a machine where no other user has `sudo` rights,
because nobody will then be able to install, update or remove packages or change
the system.

**Getting root back (GRUB recovery).** If you lock yourself out:

1. Reboot and, in the **GRUB** menu, highlight the Debian entry and press `e`.
2. On the line starting with `linux /boot/vmlinuz...`, replace `ro quiet` with
   `rw init=/bin/bash`.
3. Press `Ctrl+X` to boot. A root shell opens without a password.
4. Run:

        mount -o remount,rw /
        passwd -u root
        usermod -s /bin/bash root
        passwd root
        sync
        reboot -f

`passwd root` asks for a new password.

![passwd -S root, locked](screenshots/root-locked.png)
![GRUB edit with init=/bin/bash](screenshots/grub-recovery.png)

### (c) Can root still log in?
No, **not with its password**: the hash is unusable, so password login fails
with an authentication error. But the account is not fully blocked. UID 0 still
exists, owns processes and services, and can be reached through `sudo -i`
or `su` from an administrator, or through SSH keys if root login is allowed. To
close those routes too, set `PermitRootLogin no` in `/etc/ssh/sshd_config`.
Root can also always be recovered with physical or console access, as shown
above.

---

## 2. PAM and NSS

**PAM (Pluggable Authentication Modules)** decides *how* users are
authenticated. Applications such as `login`, `sshd` and `sudo` call PAM, and the
policy is defined per service in `/etc/pam.d/`. Separate modules check the
password and apply security rules (account validity, session setup, password
changes) without the application knowing the details.

**NSS (Name Service Switch)** decides *where* to look up information such as
UID, GID, usernames and group memberships. The sources and their order are in
`/etc/nsswitch.conf`, for example `passwd: files ldap`.

Local accounts are stored in `/etc/passwd`, password hashes in `/etc/shadow`
and local groups in `/etc/group`. NSS reads this configuration to find account
and group data (often the local files). PAM reads its own configuration to
decide how authentication is performed, including password checks and login
rules. For LDAP both are needed: NSS makes the users **visible**, PAM lets them
**log in**.

---

## 3. OpenLDAP with PAM and NSS

### (a) Install the directory server
    sudo apt install slapd ldap-utils

When asked for the administrator password, I used `asd123` (the value given in
the assignment).

![slapd installation](screenshots/slapd-install.png)
![slapd password prompt](screenshots/slapd-password.png)

### (b) Load the preconfigured directory
I installed `curl` and ran the script from the assignment's README, which stops
the server, fetches `config.ldif` and `data.ldif`, loads them and starts the
server again. My first attempt had a typo in the URL and returned error 403.

    sudo apt install curl
    /bin/bash -c "$(curl -fsSL <SCRIPT_URL_FROM_ASSIGNMENT>)"

The preconfigured server uses: LDAP version 3, base `dc=ipb,dc=pt`, root DN
`cn=manager,dc=ipb,dc=pt`, password `asd123`.

![curl and script output](screenshots/curl-script.png)

### Check the existing users
`ldapsearch` shows the two existing students with their UIDs and their group:

    ldapsearch -LLL -x -b 'ou=users,dc=ipb,dc=pt' dn uidNumber memberUid

![ldapsearch output](screenshots/ldapsearch.png)

### (c) Add student03
LDIF (LDAP Data Interchange Format) files describe directory entries: `dn:`
first, then one `attribute: value` per line, with entries separated by a blank
line. I created `student03.ldif` with nano, copying the layout of the existing
students, and added it. The uidNumber must be unique and the gidNumber must be
the same as the other students so that the user is in the same group.

    ldapadd -x -H ldap://localhost -D "cn=manager,dc=ipb,dc=pt" -w asd123 -f student03.ldif
    ldappasswd -x -H ldap://localhost -D "cn=manager,dc=ipb,dc=pt" -W -S "<DN_OF_STUDENT03>"

The file is in [`ldif/student03.ldif`](ldif/student03.ldif). Student03 is then
added to the group's `memberUid` list (`ldapmodify`).

![student03.ldif](screenshots/student03-ldif.png)
![ldapadd and ldappasswd](screenshots/ldapadd.png)

### (d) Authenticate against LDAP, keeping local users
I installed the LDAP PAM and NSS modules. Debian replaced `libpam-ldap` and
`libnss-ldap` with `libpam-ldapd` and `libnss-ldapd`, plus the `nslcd`
daemon.

    sudo apt install libpam-ldap libnss-ldap

In the configuration screens I set the LDAP server URI to `ldap://localhost/`,
the search base to `dc=ipb,dc=pt`, and enabled the LDAP lookups for `passwd`,
`group` and `shadow`. The new source is added **last**, so local users in
`/etc/passwd` still work:

    passwd: files ldap
    group:  files ldap
    shadow: files ldap

![nslcd configuration](screenshots/nslcd-config.png)
![libnss-ldapd configuration](screenshots/libnss-config.png)

### (e) Create the home directory at first login
PAM files do not have to be edited by hand: Debian provides `pam-auth-update`.
I ticked **Unix authentication**, **LDAP Authentication** and **Create home
directory on login**, which adds `pam_mkhomedir.so` to the session stack.

    sudo pam-auth-update

![pam-auth-update](screenshots/pam-auth-update.png)

### (f) Show that LDAP users and groups are visible
    getent passwd student03
    id student03
    getent group students
    ldapsearch -x -LLL -H ldap://localhost -b dc=ipb,dc=pt dn

`grep student03 /etc/passwd` returns nothing, so the user comes from LDAP and
not from the local files.

![getent and id](screenshots/getent-id.png)
![ldapsearch, all entries](screenshots/ldapsearch-all.png)

### (g) Log in as the three students
    su - student03
    su - student01
    su - student02

Each login created the home directory (`Creating directory '/home/studentNN'`)
and printed the right working directory. All passwords are `asd123`.

![su logins](screenshots/su-logins.png)

---

## 4. Dual boot: Debian and Windows

The task: a VM with enough disk space for Windows and Debian to coexist,
installing **Linux first**, then **Windows**, with the boot menu belonging to
Linux and also
