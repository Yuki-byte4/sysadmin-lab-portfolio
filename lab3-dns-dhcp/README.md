# Lab 3: DNS and DHCP (BIND9 and ISC DHCP)

**Environment:** Debian 13 (trixie) servers and an Ubuntu Desktop workstation in VMware Workstation, NAT network

## Contents
1. [Theory](#1-theory)
2. [Network layout](#2-network-layout)
3. [BIND9: primary and secondary DNS](#3-bind9-primary-and-secondary-dns)
4. [ISC DHCP: main and failover](#4-isc-dhcp-main-and-failover)
5. [Workstation tests](#5-workstation-tests)
6. [Failover tests](#6-failover-tests)
7. [Problems and fixes](#7-problems-and-fixes)

---

## 1. Theory

### BIND9
BIND9 is a flexible, full-featured DNS (Domain Name System) server. It can act
as a caching server that resolves queries recursively, and as an
**authoritative** server, meaning the server that holds the official records
for a domain and gives the definitive answer about its addresses. Information
for a domain is organised into **zones**.

- **Primary zone:** stores the original records and is modified only by the
  administrator.
- **Secondary zone:** stores a read-only copy of the primary zone and is updated
  through **zone transfers**.

### DHCP and ISC DHCP
DHCP (Dynamic Host Configuration Protocol) assigns IP addresses and network
settings to devices so they can communicate on a network. ISC DHCP
(`dhcpd`) is a software suite that implements all aspects of DHCP.

---

## 2. Network layout

Five VMs on the same NAT network (VMnet8):

| Host | Role | Address |
|---|---|---|
| l3s1h-dns1 | DNS primary | `192.168.124.200` |
| l3s1h-dns2 | DNS secondary | `192.168.124.201` |
| l3s1h-dhcp1 | DHCP main | `192.168.124.100` |
| l3s1h-dhcp2 | DHCP failover | `192.168.124.101` |
| l3s1h-wstation | Ubuntu workstation | `192.168.124.90` (reserved) |

First I checked the automatically configured subnet, netmask, gateway and DNS
server on one host. This proved that the NAT network's DHCP was working. The
task then required static addresses on the four servers, so I changed them
manually. The subnet found here is `192.168.124.0/24`, gateway `192.168.124.2`.

VMware's own DHCP service on VMnet8 must be disabled so it does not answer
instead of our server (Virtual Network Editor → VMnet8 → untick *Use local DHCP
service*). It must be **re-enabled** after the lab, or later VMs will not get
addresses.

![Auto-configured network values](screenshots/01-auto-config.png)
![Static addresses](screenshots/02-static-ip.png)
![VMnet8 DHCP disabled](screenshots/03-vmnet8-dhcp.png)

---

## 3. BIND9: primary and secondary DNS

### Install and permissions
BIND9 and its utilities were installed on dns1 and dns2. The group `bind` must
be able to write in `/etc/bind`: the secondary needs it to save zone files, and
the primary needs it for dynamic updates.

    apt install bind9 bind9-utils dnsutils
    chgrp bind /etc/bind
    chmod g+w /etc/bind

### Configuration files
- `named.conf.options` holds global settings (listening addresses, recursion,
  forwarders, who may transfer zones).
- `named.conf.local` defines the zones this server manages: type (primary or
  secondary), zone file location and transfer settings.

A forward zone `linux.local` and a reverse zone `124.168.192.in-addr.arpa` were
created with records for all VMs except the workstation (its record is created
dynamically, see DDNS).

![named.conf.options](screenshots/04-named-options.png)
![named.conf.local on dns1](screenshots/05-named-local-dns1.png)
![Forward zone file](screenshots/06-zone-forward.png)
![Reverse zone file](screenshots/07-zone-reverse.png)

### DDNS key
A DDNS key is a shared secret that lets the DHCP server update DNS records
automatically. Not every device may change DNS records, so DNS only accepts
updates signed with this key. It works like a password shared between DHCP and
DNS, similar to symmetric cryptography.

    tsig-keygen -a hmac-sha256 ddns-key > /etc/bind/ddns.key
    chown bind:bind /etc/bind/ddns.key
    chmod 640 /etc/bind/ddns.key

The key is included from `named.conf.local` with
`include "/etc/bind/ddns.key";`. (The secret is hidden in the screenshots.)

### Check and start
    named-checkconf
    named-checkzone linux.local /etc/bind/db.linux.local
    named-checkzone 124.168.192.in-addr.arpa /etc/bind/db.192.168.124
    systemctl restart named
    dig @localhost l3s1h-dhcp1.linux.local +short

![Config and zone checks](screenshots/08-checks.png)
![dig on dns1](screenshots/09-dig-dns1.png)

### Secondary server (dns2)
dns2 was set up like dns1 (installation, permissions), with `named.conf.local`
declaring both zones as `type secondary;` and listing dns1 in `primaries`.
There is no `allow-transfer` on dns2, because it only receives zones.

![named.conf.local on dns2](screenshots/10-named-local-dns2.png)

### Problem: the zone files were not saved on dns2
The log showed the transfers succeeded, but each save failed:

    transfer of 'linux.local/IN' from 192.168.124.200#53: Transfer status: success
    dumping master file: /etc/bind/tmp-...: open: permission denied
    zone linux.local/IN: dump failed: permission denied

The zones were held in memory only, so `ls /etc/bind/db.*` showed nothing.
Giving the group `bind` write permission was not enough. The kernel messages
explain why:

    dmesg | grep -i apparmor | grep -i named
    apparmor="DENIED" operation="mknod" profile="named" name="/etc/bind/tmp-..." requested_mask="c" fsuid=105

**AppArmor**, a mandatory access control module that works on top of normal
file permissions, blocked `named` (running as user `bind`) from creating files
in `/etc/bind`. Debian's profile for `named` normally allows only read access
there. The fix adds a rule to the local override file, which survives package
updates:

    echo '/etc/bind/** rw,' >> /etc/apparmor.d/local/usr.sbin.named
    apparmor_parser -r /etc/apparmor.d/usr.sbin.named
    systemctl restart named
    ls -l /etc/bind/db.*

Afterwards `db.linux.local` and `db.192.168.124` existed in `/etc/bind`, and no
new `DENIED` messages appeared. The same rule is needed on the primary for the
journal files of dynamic updates.

![Transfer success and dump failed](screenshots/11-dump-failed.png)
![AppArmor DENIED in dmesg](screenshots/12-apparmor-denied.png)
![Files created after the fix](screenshots/13-db-files.png)

---

## 4. ISC DHCP: main and failover

### Install and interface
On dhcp1 and dhcp2:

    apt install isc-dhcp-server

It may fail to start before configuration, which is normal. The interface is set
in `/etc/default/isc-dhcp-server`:

    INTERFACESv4="ens33"

### dhcp1 configuration
The configuration file is **`/etc/dhcp/dhcpd.conf`** (not `dhcp.conf`, which
the server ignores). It has these sections:

- global options: domain name, DNS servers, lease times
- **DDNS settings:** the key and the `zone` blocks pointing to dns1
- **failover peer** for dhcp2
- the **network (subnet)** with the pool and range `.10` to `.50`
- a **reservation** for the workstation, using its MAC address
  (`ip link` on the Ubuntu VM) and fixed address `192.168.124.90`

![dhcpd.conf on dhcp1](screenshots/14-dhcpd-conf.png)
![isc-dhcp-server running](screenshots/15-dhcp-status.png)

### dhcp2 (failover)
dhcp2 uses the same `dhcpd.conf` (same key, zones, subnet, pool and host
reservation) with a different failover block. Two parameters exist only on the
primary:

- **`mclt`**: the maximum time one server may extend a client's lease beyond
  what the other server knows about (3600 seconds here).
- **`split`**: how the two servers divide the clients. `split 128` is 50/50, so
  both servers share the work almost equally.

![Failover block on dhcp2](screenshots/16-failover-dhcp2.png)

---

## 5. Workstation tests

On Ubuntu Desktop the wired connection was set to automatic (DHCP) for IPv4 and
DNS, then reconnected. The checks show the address `192.168.124.90/24`, gateway
`192.168.124.2`, the two DNS servers and the domain `linux.local`. Name
resolution was then tested with `nslookup` for the DHCP servers, DNS server,
workstation, a reverse lookup and an external name.

![Workstation network settings](screenshots/17-ws-network.png)
![nslookup results](screenshots/18-nslookup.png)

---

## 6. Failover tests

### DNS: stop dns1
On dns1: `systemctl stop named`. The same queries were repeated against dns2,
which answered from its copy of the zones.

![dns1 stopped, dns2 answering](screenshots/19-dns-failover.png)

### DHCP: stop dhcp1
With dhcp1 stopped, the workstation renewed its lease. The line
`DHCPACK of 192.168.124.90 from 192.168.124.101` proves that dhcp2 answered.

    sudo dhclient -r
    sudo dhclient -v

![DHCPACK from dhcp2](screenshots/20-dhcp-failover.png)

---

## 7. Problems and fixes

| Problem | Cause | Fix |
|---|---|---|
| `apt` could not resolve `security.debian.org` | Static IP without a DNS server | Add `nameserver` to `/etc/resolv.conf` or `dns-nameservers` to the interface |
| `named` would not start | `allow-update` set twice, and typos (`listens-on`, `allow-transfers`) | Run `named-checkconf` and fix each line |
| `named-checkzone`: file not found | Zone file not created, and a path typo | Create the zone files, use `/etc/bind/db...` |
| `tsig-keygen: command not found` | `/usr/sbin` not in PATH | Run `/usr/sbin/tsig-keygen` or use `su -` |
| Secondary did not save zones | AppArmor allows `named` only to read `/etc/bind` | Add `/etc/bind/** rw,` to the local AppArmor override |
| PC could not reach the VMs | VMnet8 adapter had a `169.254.x.x` address after disabling VMware DHCP | Set `192.168.124.1/24` on the VMnet8 adapter |
| DHCP server ignored its config | File named `dhcp.conf` instead of `dhcpd.conf` | Use `/etc/dhcp/dhcpd.conf` |

## What I learned
- A secondary DNS server needs write access for its zone files, and security
  modules such as AppArmor apply on top of file permissions.
- Dynamic DNS needs a shared TSIG key between the DHCP and DNS servers.
- Failover shares the lease pool between two servers, so one can answer when the
  other is down.
- Disable VMware's own DHCP for this lab, and re-enable it afterwards.
